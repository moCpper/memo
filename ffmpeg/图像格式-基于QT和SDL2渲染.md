# RGB格式
****

- RGB的定义
  -   RGB888 是一种颜色表示方法，使用 24 位（8 位红色、8 位绿色和 8 位蓝色）来表示颜色。其他RGB变体如RGB565则依此类推，使用 16 位表示颜色（5 位红色，6 位绿色，5 位蓝色）
  -   ARGB888- 是一种扩展的颜色表示方法，使用 32 位（8 位 alpha 通道、8 位红色、8 位绿色和 8 位蓝色）来表示颜色。

| 属性  | RGB888        | ARGB888            |
| --- | ------------- | ------------------ |
| 位数  | 24            | 32                 |
| 通道数 | 3通道(R G B)    | 4通道(A R G B)       |
| 透明度 | 不支持           | 支持                 |
| 用途  | 通常用于不需要透明度的图像 | 适用于需要透明度的图像（如图层叠加） |

```cpp
void TestRGB::paintEvent(QPaintEvent* ev){
    QImage im(w, h, QImage::Format_RGB888);         //RGB888表示每24bit组成一个像素
    auto d = im.bits();                             //QImage::bits() 返回第一个像素地址
    int r = 255;
    for (int j = 0; j < h; j++) {
        int b = j * w * 3;                          //获取每行首像素的偏移
        r--;
        for (int i = 0; i < w * 3; i += 3) {
            d[b + i] = r;
            d[b + i + 1] = 0;
            d[b + i + 2] = 0;
        }   
    }

    QPainter p;
    p.begin(this);
    p.drawImage(0, 0, im);                          //0，0坐标从屏幕左上角开始计算
    p.end();
}

```

---
QImage和QPainter为QT绘图类。

-QImage提供图像的像素级的操作,其中QImage::bits()return图片首个'像素'的地址。
-QPainter则负责在绘图设备上进行绘图操作，用以将QImage修改的图片在窗口上进行绘制。
>
*override窗口的paintEvent以达到在自定义控件或窗口上进行绘图操作,其在调用show()时会被自动调用。*

---
绘制结果如图：
![[Pasted image 20240805152130.png]]


# SDL
****
基于SDL的图像渲染，以下实现了一个红色的渐变渲染：
```cpp
#include <iostream>
extern "C"{
#include<SDL.h>
}
#pragma comment(lib,"SDL2.lib")
#undef main
int SDL_main(int argc,char* argv[]){
    int w = 800;
    int h = 600;
    //1初始化SDL video库
    if (SDL_Init(SDL_INIT_VIDEO)) {
        std::cout << SDL_GetError();
        return -1;
    }

    //2生成SDL窗口
    auto screen = SDL_CreateWindow("test sdl ffmpeg", 
        SDL_WINDOWPOS_CENTERED,                 //窗口位置
        SDL_WINDOWPOS_CENTERED,
        w,h,
        SDL_WINDOW_OPENGL|SDL_WINDOW_RESIZABLE);
    if (!screen) {
        std::cout << SDL_GetError() << std::endl;
        return -1;
    }

    //3生成渲染器
    auto render = SDL_CreateRenderer(screen, -1,SDL_RENDERER_ACCELERATED);
    if (!render) {
        std::cout << SDL_GetError() << std::endl;;
        return -1;
    }

    //4生成材质
    auto texture = SDL_CreateTexture(render, SDL_PIXELFORMAT_ARGB8888,
        SDL_TEXTUREACCESS_STREAMING, w, h);                                         //可加锁
    if (!texture) {
        std::cout << SDL_GetError() << std::endl;
        return -1;
    }
    unsigned char _r = 255;
    //存放图像的数据
    unsigned char* r = new unsigned char[w * h * 4];
    for (;;) {
        _r--;
        SDL_Event ev;
        SDL_WaitEventTimeout(&ev,10);
        if (ev.type == SDL_QUIT) {
            SDL_DestroyWindow(screen);
            break;
        }

        for (int j = 0; j < h; j++) {
            int b = j * w * 4;
            for (int i = 0; i < w * 4; i += 4) {
                r[b + i] = 0;                   //B
                r[b + i + 1] = 0;               //G
                r[b + i + 2] = _r;             //R
                r[b + i + 3] = 0;               //A
            }
        }

        //5内存数据写入材质
        SDL_UpdateTexture(texture, NULL, r, w * 4);

        //6清理屏幕
        SDL_RenderClear(render);
        SDL_Rect sdl_rect;
        sdl_rect.x = 0;
        sdl_rect.y = 0;
        sdl_rect.w = w;
        sdl_rect.h = h;

        //7复制材质到渲染器
        SDL_RenderCopy(render, texture,
            NULL,                               //原图位置和尺寸
            &sdl_rect);                         //目标位置和尺寸

        //8渲染
        SDL_RenderPresent(render);
    }
    delete[] r;
    ::getchar();
    return 0;  
}
```


**SDL_CreateWindow()**
-  用于创建一个SDL窗口，提供给你绘制图形和处理用户输入的区域。
```cpp
auto screen = SDL_CreateWindow("test sdl ffmpeg", 
        SDL_WINDOWPOS_CENTERED,                 //窗口位置
        SDL_WINDOWPOS_CENTERED,
        w,h,
        SDL_WINDOW_OPENGL|SDL_WINDOW_RESIZABLE);
```
其传入的参数分别为 ： 
   -  窗口标题
   -  窗口水平位置
   -  窗口垂直位置
      -  **`SDL_WINDOWPOS_CENTERED`**: 将窗口的位置设置为屏幕中心。这是一个常用的标志，表示窗口在屏幕上的水平和垂直位置都居中。
   -  窗口宽度和高度
   - 以及窗口标志
      - **`SDL_WINDOW_OPENGL`**: 指示窗口将使用 OpenGL 上下文。这通常用于需要进行 OpenGL 渲染的应用程序。
      - **`SDL_WINDOW_RESIZABLE`**: 允许用户调整窗口的大小。这使得窗口可以被拖动和改变尺寸，提供更大的灵活性。

 **SDL_CreateRenderer()**
- 其函数返回一个渲染器，这个渲染器可以用来在窗口上绘制图形，是渲染图像和图形的主要工具，在使用 SDL 进行图形绘制时是不可或缺的。
- 参数分别为为关联的窗口，使用默认的渲染驱动，以及使用硬件加速
```cpp
  auto render = SDL_CreateRenderer(screen, -1,SDL_RENDERER_ACCELERATED);
    if (!render) {
        std::cout << SDL_GetError() << std::endl;;
        return -1;
    }
```

**SDL_CreateTexture**
- 用于创建纹理（texture）的函数。纹理是一种图像资源，它用于在渲染过程中将图像绘制到窗口上。这个函数的作用和参数解释如下：
    -   要将纹理与之关联的渲染器传入。纹理会通过这个渲染器进行渲染
    -   纹理的像素格式。定义了纹理中每个像素的颜色信息的布局和深度。例如`SDL_PIXELFORMAT_ARGB8888`
    -   纹理的访问模式。执行策略如下：
	      - `SDL_TEXTUREACCESS_STATIC`: 纹理内容在创建时设置且不再改变。
		  - `SDL_TEXTUREACCESS_STREAMING`: 纹理内容可以在运行时动态更新。
		  - `SDL_TEXTUREACCESS_TARGET`: 纹理可以被用作渲染目标（即，渲染到这个纹理上。
	- 宽高
```cpp
  auto texture = SDL_CreateTexture(render, SDL_PIXELFORMAT_ARGB8888,
        SDL_TEXTUREACCESS_STREAMING, w, h);    
```

**SDL_UpdateTexture**
- 是 SDL 库中用于更新纹理内容的函数。它的主要作用是在运行时将内存中的图像数据复制到指定的纹理中，以便后续的渲染操作使用。

```cpp
SDL_UpdateTexture(texture, NULL, r, w * 4);
```

**SDL_RenderClear**
- 清空指定的渲染器 (`render`) 的当前内容。这一行会用渲染器的当前绘制颜色填充整个窗口，通常是黑色或其他设置的颜色。
- 在每帧渲染开始时，首先清除上一次的绘制内容，以避免在屏幕上留下残影。
```cpp
SDL_RenderClear(render);
```

**SDL_RenderCopy**
- 将 `texture` 纹理中的图像拷贝到渲染器中，并按照 `sdl_rect` 指定的位置和大小进行绘制。
- 第三个参数指定源texture的范围以拷贝到render。
- 第四个参数指定目标绘制区域的信息和尺寸。
```cpp
SDL_Rect sdl_rect;   //用于指定目标绘制区域的位置信息和尺寸。
sdl_rect.x = 0;
sdl_rect.y = 0;
sdl_rect.w = w;
sdl_rect.h = h;
SDL_RenderCopy(render,texture,NULL,&sdl_rect);       
```

**SDL_RenderPresent**
- 将当前渲染器的内容呈现到窗口上，显示所有通过 `SDL_RenderCopy` 绘制的内容。这一行实际上将之前在渲染器中进行的所有绘制操作（比如清空、绘制纹理等）更新到屏幕上，确保用户能够看到最新的渲染结果。
```cpp
SDL_RenderPresent(render);
```

在QT中渲染控件,效果如上。
```cpp
static SDL_Window* sdl_win = nullptr;
static SDL_Renderer* sdl_render = nullptr;
static SDL_Texture* sdl_texture = nullptr;
static int sdl_w = 0;
static int sdl_h = 0;
static unsigned char* rgb = nullptr;
static int pix_size = 4;
static unsigned char t = 255;

TestRGB::TestRGB(QWidget *parent)
    : QWidget(parent){

    ui.setupUi(this);
    int w = this->width();
    int h = this->height();
    ui.label->setFixedSize(QSize(w,h));
    sdl_w = ui.label->width();
    sdl_h = ui.label->height();

    SDL_Init(SDL_INIT_VIDEO);

    sdl_win = SDL_CreateWindowFrom((void*)ui.label->winId());

    sdl_render = SDL_CreateRenderer(sdl_win, -1, SDL_RENDERER_ACCELERATED);

    sdl_texture = SDL_CreateTexture(sdl_render,SDL_PIXELFORMAT_ARGB8888,SDL_TEXTUREACCESS_STREAMING,sdl_w,sdl_h);

    rgb = new unsigned char[sdl_w*sdl_h*4];
    startTimer(10);
}

void TestRGB::timerEvent(QTimerEvent* ev) {
    t--;
    for (int j = 0; j < sdl_h; j++) {
        int b = j * sdl_w * pix_size;
        for (int i = 0; i < sdl_w*pix_size; i += pix_size) {
            rgb[b + i] = 0;
            rgb[b + i + 1] = t;
            rgb[b + i + 2] = 0;
            rgb[b + i + 3] = 0;
        }
    }
    SDL_UpdateTexture(sdl_texture,NULL,rgb,sdl_w*pix_size);
    SDL_RenderClear(sdl_render);
    SDL_Rect rect{};
    rect.x = 0;
    rect.y = 0;
    rect.w = sdl_w;
    rect.h = sdl_h;
    SDL_RenderCopy(sdl_render, sdl_texture,NULL,&rect);
    SDL_RenderPresent(sdl_render);
}
```

# 使用SDL渲染合并两幅图像
****

```cpp
TestRGB::TestRGB(QWidget *parent)
    : QWidget(parent){

    ui.setupUi(this);

    SDL_Init(SDL_INIT_VIDEO);

    sdl_win = SDL_CreateWindowFrom((void*)ui.label->winId());

    sdl_render = SDL_CreateRenderer(sdl_win, -1, SDL_RENDERER_ACCELERATED);

    QImage img1("1.png");
    QImage img2("2.png");
    if (img1.isNull() || img2.isNull()) {
        QMessageBox::information(this,"open image error","open image filed！");
        return;
    }
    int out_w = img1.width() + img2.width();
    int out_h = img1.height() > img2.height() ? img1.height() : img2.height();

    sdl_w = out_w;
    sdl_h = out_h;
    resize(sdl_w, sdl_h);
    ui.label->move(0, 0);
    ui.label->resize(sdl_w, sdl_h);

    sdl_texture = SDL_CreateTexture(sdl_render,SDL_PIXELFORMAT_ARGB8888,SDL_TEXTUREACCESS_STREAMING,sdl_w,sdl_h);

    rgb = new unsigned char[sdl_w*sdl_h*4];

    //默认设置为透明
    ::memset(rgb, 0, sdl_w * sdl_h * pix_size);

    //合并两幅图像
    for (int i = 0; i < sdl_h; i++) {
        int b = i * sdl_w * pix_size;
        if (i < img2.height()) {
            ::memcpy(rgb+b,img2.scanLine(i),img2.width()*pix_size);
        }
        b += img2.width() * pix_size;
        if (i < img1.height()) {
            ::memcpy(rgb+b, img1.scanLine(i), img1.width() * pix_size);
        } 
    }

    QImage out(rgb,sdl_w,sdl_h,QImage::Format_ARGB32);
    out.save("out.png");

    startTimer(10);
}
```

渲染后如图:
![[Pasted image 20240806143617.png]]

# YUV格式
****

 YUV 中Y是指亮度分量，U指蓝色色度分量，而V指红色色度分量。
 
YUV和RGB的主要区别

|             | YUV                              | RGB                         |
| ----------- | -------------------------------- | --------------------------- |
| **色彩表示方式**: | 将颜色信息分为亮度和色度，适用于视频压缩和传输。         | 直接表示颜色分量（红、绿、蓝），适用于图像处理和显示。 |
| **数据量和压缩**  | 通过降低色度分量的采样率来减少数据量，提高压缩效率        | 每个像素需要三个分量（R、G、B），通常数据量较大。  |
| **视觉效果**    | 更适合视频处理和传输，能够在降低数据量的同时保持较高的视觉质量。 | 更适合需要高色彩精度的场景，如图像编辑和计算机显示   |
| **用途**      | 常用于视频压缩、广播电视和视频编码。               | 通常用于计算机图像处理、显示器显示。          |

由于人眼对 Y 的敏感度远超于对 U 和 V 的敏感，所以有时候可以多个 Y 分量共用一组 UV，这样既可以极大得节省空间，又可以不太损失质量。
所以YUV被分为了三种格式，分别为: **YUV 420**，**YUV 422**，**YUV 444**,其中4表示4个像素点为一组
- YUV 420，由 4 个 Y 分量共用一套 UV 分量，
	-  在色彩精度和数据量之间取得平衡，适合视频采集和传输。
- YUV 422，由 2 个 Y 分量共用一套 UV 分量
	-  在色彩精度和数据量之间取得平衡，适合视频采集和传输。
- YUV 444，不共用，一个 Y 分量使用一套 UV 分量
    -   提供最高的色彩精度和数据量，适合高质量需求

**相比于RGB的交叉存储,例如:RGB RGB RGB RGB，YUV为平面存储:如:yyyyyyyy uu vv**
例如：
![[Pasted image 20240808122058.png]]

使用ffmpeg.exe 将mp4转为yuv
![[Pasted image 20240806203245.png]]
 根据提供的日志 **Stream #0:0(eng): Video: rawvideo (I420 / 0x30323449), yuv420p(progressive), 400x300 [SAR 4:3 DAR 16:9], q=2-31, 36000 kb/s, 25 fps, 25 tbn (default)**
 可见，将mp4转换为了yuv420p,25帧。
 
示例: 使用SDL_QT播放渲染YUV
```cpp
static SDL_Window* sdl_win = nullptr;
static SDL_Renderer* sdl_render = nullptr;
static SDL_Texture* sdl_texture = nullptr;
static int sdl_w = 0;
static int sdl_h = 0;
static unsigned char* yuv = nullptr;
static int pix_size = 2;
static std::ifstream yuv_file;

TestRGB::TestRGB(QWidget *parent)
    : QWidget(parent){

    //打开yuv
    yuv_file.open("400_300_25.yuv",std::ios::binary);
    if (!yuv_file) {
        QMessageBox::information(this,"open error","open yun error");
        return;
    }

    ui.setupUi(this);

    SDL_Init(SDL_INIT_VIDEO);

    sdl_win = SDL_CreateWindowFrom((void*)ui.label->winId());

    sdl_render = SDL_CreateRenderer(sdl_win, -1, SDL_RENDERER_ACCELERATED);

    int out_w = 400;
    int out_h = 300;

    sdl_w = out_w;
    sdl_h = out_h;

    sdl_texture = SDL_CreateTexture(sdl_render,
        SDL_PIXELFORMAT_IYUV,                       
        SDL_TEXTUREACCESS_STREAMING,
        sdl_w,sdl_h);

    yuv = new unsigned char[sdl_w*sdl_h*pix_size];

    startTimer(10);
}

void TestRGB::timerEvent(QTimerEvent* ev) {

    yuv_file.read((char*)yuv, sdl_w*sdl_h*1.5);

    SDL_UpdateTexture(sdl_texture,NULL,yuv,sdl_w);
    SDL_RenderClear(sdl_render);
    SDL_Rect rect{};
    rect.x = 0;
    rect.y = 0;
    rect.w = sdl_w;
    rect.h = sdl_h;
    SDL_RenderCopy(sdl_render, sdl_texture,NULL,&rect);
    SDL_RenderPresent(sdl_render);
}
```

注意:
因为YUV 4:2:0 中 每4个Y分量对应2个UV分量，故而U 和 V 分量的内存大小为 `(sdl_w / 2) * (sdl_h / 2)` 字节
推到出总字节数为sdl_w * sdl_h + (sdl_w / 2) * (sdl_h / 2)  + (sdl_w / 2) * (sdl_h / 2)
得到`sdl_w * sdl_h * 1.5`。
```cpp
yuv_file.read((char*)yuv, sdl_w*sdl_h*1.5);
```

在 IYUV 格式中，每个 Y 像素对应 1 字节，而 UV 分量的数量和排列方式是按行的，但它们通常会在后续处理阶段进行解码或转换，所以这里传递 Y 分量的宽度足够了。
```cpp
SDL_UpdateTexture(sdl_texture,NULL,yuv,sdl_w);
```