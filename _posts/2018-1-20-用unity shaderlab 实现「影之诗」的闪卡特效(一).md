
## 引言

大家好我是9级铁甲蛹，我从今天开始写博客了。虽然之前总想过要写写技术博客，但是觉得自己之前的学习经验对大家可能没什么帮助，而且网上许多资料非常详细。现在决定写一来是因为最近做了些有意思的东西，恰巧网上没什么具体的内容。二来是因为假期到了没之前那么忙，可以静下心来总结一下之前的学习。希望大家有什么问题多多交流。


##「影之诗」中的闪卡特效
去年cygames在unite2017 Tokyo上分享了他们制作「影之诗」的经历，其中专门讲了他们是如何制作卡牌特效的。对相关的具体内容感兴趣的朋友可以看看[游戏葡萄](http://youxiputao.com/articles/12060)和[旅法师营地](http://www.iyingdi.cn/web/article/search/44753?seed=17)的文章。
其实本质上就是一个有多种动画效果的shader，只是他们用一张图的rgb三个通道，分别作为不同效果的遮罩层，并且详细指定了各通道实现的效果类型。这样既能通过对遮罩层的详细绘制来控制效果的细节，又能通过shaderlab的便利在unity编辑器里通过调节不同参数实现不同效果。
<img src="http://wx1.sinaimg.cn/mw690/006nZWNtgy1fdgwdb6dvbg30hj0mw4qx.gif" height="400px" /> <img src="http://wx1.sinaimg.cn/mw690/006nZWNtgy1fdhucthy6gg30gp0mke86.gif" height="400px" />

「影之诗」中闪卡的效果大致分为三种(毕竟他们只用一张图的三个通道做遮罩层)。 

 1. 图中物体各种动感的效果，比如图中人物的头发、衣物的飘动效果。这些部分都是图中原有的部分，它们基本在一定的范围内运动。
 2. 在原图上添加的有动态效果的部分，比如左图中龙口中的火焰，图中上半部分的气流，以及右图中的飘动的雾气，这些是原图中没有，是在原图基础上混合进去的部分，大多伴有简单的运动规律，如平移、旋转等。
 3. 是图片中的部分区域存在的颜色变化，比如右图中的弓上的流光。
 
接下来我就这三种类型的效果详细展示各效果的实现思路，效果和代码。

##卡牌上的波动效果
报道的译文中提到他们可以选择“滚动”、“回旋”、“极坐标”等移动方式，刚看到这三词时我也没想到到底它们指那些变化方式，尤其是“滚动”，后来看到OpenCanvas7里滤镜变形里就三个选项“旋涡”“波浪”“极坐标”大概就猜到是指什么了。对应PS中的就是波浪、旋转扭曲、和极坐标这些滤镜的效果。实际上“滚动”跟PS里的“波浪”不太一样，不过算法只差一点。讲的时候我会将“波浪”和“滚动”两种效果对比展示一下。
### 波浪、滚动
无论是波浪还是滚动，这一系列的效果都可以看做是通过周期函数实现图片上像素显示颜色的周期变化。具体实现都是通过给每一个片元的uv增加一个不同的偏移量来实现的。
这个偏移量是有周期性的，同时由于效果的动态性，它是跟时间变量相关。要实现不同位置的片元可以计算出不同的偏移量，我们还要将原有的UV纳入考量，可以用原有的UV来作为周期函数的初相。
决定是波浪效果还是滚动效果的是作为初相的UV是直接作用到自己的方向上，还是和将UV的顺序颠倒分别作用到y和x方向上。
代码如下:

``` glsl
fixed3 mask = tex2D(_Mask,i.uv).rgb;
float2 phase = (i.uv+_Speed*float2(_Time.y,_Time.y))*pi*2;//原UV作为初相，为了便于描述数据和效果，再乘上2π
float2 offset;
//使用不同参数控制不同方向上的振幅、频率。
#if _TYPE_ROLL
	offset.x = _Parameters.x*sin(_Parameters.y*phase.x);
	offset.y = _Parameters.z*sin(_Parameters.w*phase.y);
#elif _TYPE_WAVE
	offset.y = _Parameters.x*sin(_Parameters.y*phase.x);
	offset.x = _Parameters.z*sin(_Parameters.w*phase.y);
#endif
fixed4 col = tex2D(_MainTex, i.uv+0.001*offset*mask.b);
```
以下图片都使用了(4,4,4,4)作为_Parameters,_Speed为0.25。可以看到两种不同的计算方式的差别，波浪方式就像它的名字一样可以在图片上看到图片的波动，而滚动则像是图片局部拉大又拉小的过程。
![两种不同位移方式的对比](https://l9metapod.github.io/assets/img/other/nomask.gif)

如果把一个方向上的振幅调为0，那么滚动效果的图片就像一张随风飘扬的旗子。这样看应该就明白我为什么猜测后一种的效果就是cygames所使用的“滚动”，从一个方向上看它就像一块布后面有一排圆柱体从它身上滚过去一样。
![一个方向的振幅为0时两种不同位移方式的对比](https://l9metapod.github.io/assets/img/other/2018-01-20_18-04-59.gif)

然后就是应用上我们的遮罩层了，代码里我用了图片的b通道保存这个效果的信息，所以遮罩层的蓝色颜色越大图片就抖动的程度就越大。
这里讲一个小技巧，虽然遮罩图让人来绘制以控制细节效果更好，但我们只是简单实现一下效果可以对原图进行一些简单处理来得到遮罩图。我们可以先在PS中扣出想要动的区域，然后将其转为黑白图，如果黑白图中更亮的部位是动的程度更高的部分那就不再改变图片，直接用它来作遮罩图。如果更亮的部分是动的更小的区域，那就用它的灰度翻转图来作遮罩层。
![这里写图片描述](http://img.blog.csdn.net/20180120184012732?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFubmFuMDgxMTY2Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

应用遮罩层后的效果
![应用遮罩层后两种不同位移方式的对比](https://l9metapod.github.io/assets/img/other/mask.gif)
可以看到虽然应用在整张图时滚动的效果很迷，但是如果圈定了应用范围和强度，滚动的效果有很强的表现力，可以更好的体现纵深方向的变化。
最后附上完整源代码
``` glsl
Shader "Custom/SVCardEffect"
{
	Properties
	{
		
		_MainTex ("Texture", 2D) = "white" {}
		[NoScaleOffset]
		_Mask("Mask", 2D) = "white" {}
		_Speed("Speed", Range(0,10)) = 1
		_Parameters("Parameters", vector) = (1,1,1,1)
		[KeywordEnum(Roll,Wave)] _Type("Type",float) = 0

	}
	SubShader
	{
		Tags { "RenderType"="Opaque" }
		LOD 100

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#pragma shader_feature _TYPE_ROLL _TYPE_WAVE
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"
			#define pi 3.14159265358979 
			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _Mask;
			float4 _Mask_ST;
			float4 _Parameters;
			float _Speed;

			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				
				fixed3 mask = tex2D(_Mask,i.uv).rgb;


				float2 phase = (i.uv+_Speed*float2(_Time.y,_Time.y))*pi*2;//原UV作为初相，为了便于描述数据和效果，再乘上2π
				float2 offset;
				//使用不同参数控制不同方向上的振幅、频率。
				#if _TYPE_ROLL
					offset.x = _Parameters.x*sin(_Parameters.y*phase.x);
					offset.y = _Parameters.z*sin(_Parameters.w*phase.y);
				#elif _TYPE_WAVE
					offset.y = _Parameters.x*sin(_Parameters.y*phase.x);
					offset.x = _Parameters.z*sin(_Parameters.w*phase.y);
				#endif
				fixed4 col = tex2D(_MainTex, i.uv+0.001*offset*mask.b);

				return col;
			}
			ENDCG
		}
	}
}
```


