#水墨风格渲染
这次学校的比赛打算做一个中国古代背景的游戏，所以尝试做了水墨风格的渲染。
主要从四方面来实现的效果：

 1. 根据色调和饱和度调整饱和度。
 2. 对图像进行模糊
 3. 水墨风格的物体边缘
 4. 物体内画笔笔触的模拟

接下来我分别就这四步依次展开说明：
##调整饱和度
这里我使用Post-Processing Stack 的Color Grading 来调节饱和度，通过使用Grading Curves来具体调节。根据色调和饱和度调整饱和度。强化部分国画中常用颜料的颜色部分的饱和度，少见颜料的颜色降低饱和度；饱和度高的强化，低的降低。这里我调节了Sat VS Sat和Hue VS Sat两种曲线，保留了部分色调在红色和绿色附近（也就是所谓的丹青）的饱和度，使其它颜色整体变灰；并使饱和的较高的部分区域的饱和度更高，其余部分有所降低，效果对比如下：
![这里写图片描述](http://img.blog.csdn.net/20180309214740577?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![这里写图片描述](http://img.blog.csdn.net/20180309214748935?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
##图像模糊
这里使用高斯模糊来实现，由于与之后的描边效果不冲突可以写在一个Pass里。我这里直接用Post-Processing Stack里的DOF（景深）效果，然后把焦距调到最小，让全屏都模糊。实际上的效果是几乎一样的。
![这里写图片描述](http://img.blog.csdn.net/20180309224819413?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
##水墨描边
由于上一步我们模糊了图像，所有这里不能直接用原图来进行边缘检测，我们通过_CameraDepthNormalsTexture，来获得法线图然后用法线图来进行边缘检测，这样我们不仅得到了模糊前图的边缘而且，不会讲阴影等几何上在一个平面上的颜色不连续的部分也识别为边缘。
采用Roberts边缘算子来检测，Roberts算子计算对角上的像素之间的差，模板比较简单，而且可以通过调整中心像素到角上的距离来控制描边的粗细。
![Roberts](http://img.blog.csdn.net/20160629153544671)
具体代码如下：
``` glsl

float rgb2gray(fixed3 col){
	float gray = 0.2125 * col.r + 0.7154 * col.g + 0.0721 * col.b; 
	return gray;
}
fixed4 frag (v2f i) : SV_Target
{
	//......
	float2 texel = _MainTex_TexelSize.xy;
	//判断是否是边缘
	fixed3 col0 = tex2D(_CameraDepthNormalsTexture,i.uv+_EdgeWidth*texel*float2(1,1)).xyz;
	fixed3 col1 = tex2D(_CameraDepthNormalsTexture,i.uv+_EdgeWidth*texel*float2(1,-1)).xyz;
	fixed3 col2 = tex2D(_CameraDepthNormalsTexture,i.uv+_EdgeWidth*texel*float2(-1,1)).xyz;
	fixed3 col3 = tex2D(_CameraDepthNormalsTexture,i.uv+_EdgeWidth*texel*float2(-1,-1)).xyz;
	float edge = rgb2gray(pow(col0-col3,2)+pow(col1-col2,2));
	//.......
}
```
其中_EdgeWidth用来控制边缘粗细
理论上我们这样计算出来的edge是一个大于0小于1的很小的数，为了更好的视觉效果我们通过pow（）函数来增大edge的值，相当于进行一个对数变换，在变换后部分非边缘的区域edge也会有个较大的值，我们可以再设定一个阈值来舍弃edge不够大的部分区域。
``` glsl
edge = pow(edge,0.2);
if(edge<_Sensitive)edge=0;
return fixed4(edge,edge,edge,1.0);
```
以下为_EdgeWidth为3时，不进行任何变换、进行对数变换、设置阈值为0.35时的到的边缘图
![这里写图片描述](http://img.blog.csdn.net/20180309232537351?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

然后用edge的值来控制边缘与原图混合，可以设置具体的边缘颜色。
可以通过噪声图或噪声函数来给边缘制造水墨感。关于噪声的内容我就不具体说明了，我这里用了三维值噪声的分形叠加，为了是摄像机移动后同一个噪声纹理与模型相匹配，我通过深度值来获得像素世界坐标，再用世界坐标来生成噪声，这样噪声的样式就贴到模型上了，而不会像用屏幕坐标来采样时那样，移动屏幕会显得不自然。
``` glsl
if(edge<_Sensitive)edge=0;				
else{
	edge=noise;
}
fixed3 finalColor = (edge)*_EdgeColor.xyz+(1-edge)*col*(0.95+0.1*noise);
//非边缘部分也可以加上较小的噪声来模拟水墨在纸上扩散的质感
//因为noise在0~1的范围内所有使系数和的均值为1左右来保证画面不会更亮或更暗
return fixed4(finalColor,1.0);
```
![这里写图片描述](http://img.blog.csdn.net/20180310105113579?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
##模拟笔触
虽然通过模糊颜色上有一定的水墨扩散的感觉，通过描边可以区分模型的边缘，但图片还是有许多不自然的地方。比如颜色扩散太均匀不想“画”出来的，许多细小的边缘由于太小而形成了“噪点”。而且由于我使用的噪声是通过时间坐标得到的，部分很远的地方由于透视而显得噪声很乱。所以我们再进行一次滤波来消除这些问题，并模拟画笔的笔触。这里用了一种保留边缘的滤波，算法具体的名字我忘了，之前不记得哪里看到过。具体思路是用中心像素作为模板的四个角，然后分别计算中心在四个角时的整个区域的均值和方差，然后取方差最小的区域的均值输出。求知道这个滤波名字的大佬告知一下，我好查一查相关资料，浅墨的[这篇文章](http://blog.csdn.net/poem_qianmo/article/details/49719247)里实现的油画效果也是用简化后，只取两个角来计算的。我这里就直接用两个角的来算了
``` glsl
for (int j = -4; j <= 0; ++j)  
{  
    for (int k = -4; k <= 0; ++k)  
    {  
        c = tex2D(_MainTex, i.uv +texel*float2(k, j)).xyz;
        m0 += c;   
        s0 += c * c;  
    }  
}
for (int j = 0; j <= 4; ++j)  
{  
    for (int k = 0; k <= 4; ++k)  
    {  
        c = tex2D(_MainTex, i.uv +texel*float2(k, j)).xyz;
        m1 += c;   
        s1 += c * c;  
    }  
}
//取方差小的区域的颜色作为最终输出颜色  
float4 finalFragColor = 0.;  
float min_sigma2 = 1e+2;                   
m0 /= 25;  
s0 = abs(s0 / 25 - m0 * m0);  
float sigma2 = s0.r + s0.g + s0.b;  
if (sigma2 < min_sigma2)   
{  
    min_sigma2 = sigma2;  
    finalFragColor = float4(m0, 1.0);  
}                     
m1 /= 25  ;
s1 = abs(s1 / 25 - m1 * m1);    
sigma2 = s1.r + s1.g + s1.b;  
if (sigma2 < min_sigma2)   
{  
    min_sigma2 = sigma2;  
    finalFragColor = float4(m1, 1.0);  
}
return finalFragColor;
```
注意这里是对描边后的图像进行的操作，最终效果如下![这里写图片描述](http://img.blog.csdn.net/20180310113440772?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
可以调节最开始的滤波范围和描边粗细，边缘颜色等参数来调整效果
![这里写图片描述](http://img.blog.csdn.net/20180310114014497?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
![这里写图片描述](http://img.blog.csdn.net/20180310114047759?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
![这里写图片描述](http://img.blog.csdn.net/20180310114201275?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
对比后处理前原效果
![这里写图片描述](http://img.blog.csdn.net/20180310114319620?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
嗯，感觉更丑了。
