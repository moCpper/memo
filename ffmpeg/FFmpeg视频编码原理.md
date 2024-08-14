**视频编码概述**
- 像素格式转换 =》 编码 =》封装 
- 为什么要编码?
	 - 像素格式数据过大
- 原始帧AVFrame =》 压缩帧AVPacket
- 帧内压缩、帧间压缩
- 有损压缩、无损压缩

>**帧内压缩**
帧内（Intraframe）压缩也称为空间压缩（Spatial compression）。当压缩一帧图像时，仅考虑本帧的数据而不考虑相邻帧之间的冗余信息，这实际上与静态图像压缩类似。
>**帧间压缩**
帧间压缩(Interframe compression)也称为时间压缩(Temporal_compression)，是基于许多视频或动画的连续前后两帧具有很大的相关性(即连续的视频其相邻帧之间具有冗余信息)的特点来实现的；通过比较时间轴上不同帧之间的数据实施压缩，进一步提高压缩比。

#  AVCodec和AVCodecContext
- AVCodec是FFmpeg库中用于表示编解码器的抽象，用于编码或解码视频或音频流。
- 它包含了编解码器所需的所有信息，如编解码器ID、工作模式、参数等。
- AVCodec用于处理数据的编码和解码过程，例如将原始视频帧转换为压缩的比特流，或反之

>在视频处理流程中，通常首先创建一个AVFrame来存储原始视频帧，然后使用AVCodec对其进行编码，生成压缩后的数据流。解码过程则相反，从数据流中提取数据，使用AVCodec解码到AVFrame，然后再进行显示或其他处理。

AVcodec与AVCodecContext源码解析

```cpp
typedef struct AVCodec {
    /**
     * Name of the codec implementation.
     * The name is globally unique among encoders and among decoders (but an
     * encoder and a decoder can share the same name).
     * This is the primary way to find a codec from the user perspective.
     */
    const char *name;
    /**
     * Descriptive name for the codec, meant to be more human readable than name.
     * You should use the NULL_IF_CONFIG_SMALL() macro to define it.
     */
    const char *long_name;
    enum AVMediaType type;
    enum AVCodecID id;
    /**
     * Codec capabilities.
     * see AV_CODEC_CAP_*
     */
    int capabilities;
    const AVRational *supported_framerates; ///< array of supported framerates, or NULL if any, array is terminated by {0,0}
    const enum AVPixelFormat *pix_fmts;     ///< array of supported pixel formats, or NULL if unknown, array is terminated by -1
    const int *supported_samplerates;       ///< array of supported audio samplerates, or NULL if unknown, array is terminated by 0
    const enum AVSampleFormat *sample_fmts; ///< array of supported sample formats, or NULL if unknown, array is terminated by -1
    const uint64_t *channel_layouts;         ///< array of support channel layouts, or NULL if unknown. array is terminated by 0
    uint8_t max_lowres;                     ///< maximum value for lowres supported by the decoder
    const AVClass *priv_class;              ///< AVClass for the private context
    const AVProfile *profiles;              ///< array of recognized profiles, or NULL if unknown, array is terminated by {FF_PROFILE_UNKNOWN}
 
/*
....

*/
} AVCodec;
```


解析其中的各种成员变量，可得
```cpp
// 成员变量
const char *name；// 编码器名称，avcodec_find_encoder_by_name使用该变量
const char *long_name; //编码器全称
enum AVMediaType type; // 编码音视频类别
enum AVCodecID id;   // 编码器唯一id，avcodec_find_encoder使用该变量
const AVRational *supported_frames; // 支持的视频帧率
const enum AVPixelFormat *pix_fts;  // 视频像素格式
const int *supported_samplerates;  // 支持的音频采样率
const enum AVSampleFormat *sample_fmts; // 音频采样格式
const uint64_t *channel_layouts ; // 音频通道数
int priv_data_size;
const AVClass *priv_data;
struct AVCodec *next;  // 链表下一个编码器
// 成员函数
int (*send_frame)(AVCodecContext *avctx, const AVFrame *frame);    // avcodec_send_frame调用
int (*receive_packet)(AVCodecContext *avctx, AVPacket *avpkt);     // avcodec_recevie_packet调用
int (*receive_frame)(AVCodecContext *avctx, AVFrame *frame);       // avcodec_send_packet/avcodec_receive_frame调用


```
```cpp
typedef struct AVCodecContext {
    const AVClass *av_class;
    AVMediaType codec_type; //编解码器处理的媒体类型，如AVMEDIA_TYPE_VIDEO或AVMEDIA_TYPE_AUDIO
    const AVCodec *codec; // 指向当前编解码器的指针
    AVCodecID codec_id; // 编解码器的ID，例如CODEC_ID_H264
    int width, height; // 视频的情况下，宽和高
    AVRational time_base; // 时间基，用于计算时间戳
    int bit_rate; // 编码时的目标比特率
    /* 更多编解码相关的参数... */
    AVFrame *frame; // 可选的，用于存储原始数据的帧
    AVFrame *coded_frame; // 编码后的帧，解码时使用
    /* 更多功能性、控制和状态字段... */
} AVCodecContext;

```

示例: FFmpeg编码器获取和上下文打开
```cpp
auto main() -> int {
	//1 找到编码器
	auto codec = avcodec_find_encoder(AV_CODEC_ID_H264);
	if (!codec) {
		std::cerr << "codec not find ! " << std::endl;
		return -1;
	}

	//2 编码上下文
	auto c = avcodec_alloc_context3(codec);
	if (!c) {
		std::cerr << "avcodec_alloc_context3 failed ! " << std::endl;
		return -1;
	}

	//3 设定上下文参数
	c->width = 400;							//视频宽高
	c->height = 300;		

	//帧时间戳的时间单位 pts/time_base = 播放时间(秒)
	c->time_base = { 1,25 };				//分数，1/25

	c->pix_fmt = AV_PIX_FMT_YUV420P;		//元数据像素格式，与编码算法相关
	c->thread_count = 16;					//编码线程数

	//4 打开编码上下文
	int re = avcodec_open2(c, codec, NULL);
	if(re != 0){
		char buf[1024]{ 0 };
		av_strerror(re, buf, sizeof(buf) - 1);
		std::cerr << "avcodec_open2 failed !" << buf << std::endl;
		return -1;
	}

	std::cout << "avcodec_open2 success!" << std::endl;

	//释放编码器上下文
	avcodec_free_context(&c);
	return 0;
}
```

**` avcodec_find_encoder `**
根据提供的编码器 ID（这里是 `AV_CODEC_ID_H264`），查找对应的编码器。如果找到了，返回一个指向 `AVCodec` 的指针；如果没有找到，返回 `NULL`。

**` avcodec_alloc_context3 `**
为编码器分配一个编码上下文 `AVCodecContext`，并初始化它。需要传入一个指向 `AVCodec` 的指针。如果成功，返回 0；如果失败，返回错误码。
- 在使用 FFmpeg 进行编解码时，首先需要通过 `avcodec_find_encoder` 或 `avcodec_find_decoder` 函数找到合适的 `AVCodec` 结构体。然后，使用 `avcodec_alloc_context3` 为特定任务分配 `AVCodecContext`，并将找到的 `AVCodec` 结构体指针赋值给 `AVCodecContext` 的 `codec` 字段。这样，`AVCodecContext` 就与特定的编解码器关联起来。`AVCodec` 提供了编解码器的描述和接口，而 `AVCodecContext` 提供了执行编解码任务所需的具体环境和参数。两者相互配合，共同完成编解码任务。

** `avcodec_open2 `**
打开编码器，准备进行编码。传入编码上下文 `AVCodecContext` 指针、编码器 `AVCodec` 指针和一个指向 `AVDictionary` 的指针（这里传 `NULL`，表示没有额外的选项）。如果成功，返回 0；如果失败，返回错误码。

**`avcodec_free_context`**:
释放之前通过 `avcodec_alloc_context3` 分配的编码上下文。传入指向 `AVCodecContext` 指针的指针。

根据输出的日志可得，所有函数执行成功，而且显示了编码器使用的 CPU 功能和编码配置的一些信息。
![[Pasted image 20240813152250.png]]


# avcodec_send_frame和avcodec_receive_packet

>编码器和解码器都维护了一个缓冲区，在刚开始输入数据时，需要多输入几帧，等缓冲区
>被填充满后，才会在receive端接收到编码或解码后的数据
>send端只把数据放入缓冲区，recive端才是解码并获得数据的函数，如果receive发现已有解码后的数据则直接获取，如果没有则开始解码

** avcodec_send_frame **
向编码器提供原始的、未压缩的音视频帧数据，以便进行压缩编码。

** avcodec_receive_frame **
向独立的编码线程中接收压缩帧。
- 参数：
	-  AVCodecContext指针，编码器上下文。用于表示编码器的配置和状态信息。
	-  AVPacket指针，数据包。用于保存接收到的编码数据。
函数返回一个整数，表示执行结果。如果返回0，代表成功；返回AVERROR(EAGAIN)，代表当前状态没有可输出数据；返回AVERROR_EOF，代表已经到达输入流结尾；返回INVAL，代表编码器没有打开或者打开的是解码器。

** av_packet_alloc **
用于分配一个 `AVPacket` 结构体的内存。`AVPacket` 用于存储编码或解码过程中的数据包。


测试用例: 创建每一帧帧数据并压缩
```cpp
//创建好AVFrame空间 未压缩数据
auto frame = av_frame_alloc();
frame->width = c->width;
frame->height = c->height;
frame->format = c->pix_fmt;
re = av_frame_get_buffer(frame, 0);
if (re != 0) {
	char buf[1024]{ 0 };
	av_strerror(re, buf, sizeof(buf) - 1);
	std::cerr << "av_frame_get_buffer failed !" << buf << std::endl;
	return -1;
}

auto pkt = av_packet_alloc();
//10秒视频 250帧
for (int i = 0; i < 250; i++) {
	//生成AVFrame数据,每帧数据不同
	//Y
	for (int y = 0; y < c->height; y++) {
		for (int x = 0; x < c->width; x++) {
			frame->data[0][y * frame->linesize[0] + x] = x + y + i * 3;
		}
	}
	//UV
	for (int y = 0; y < c->height/2; y++) {
		for (int x = 0; x < c->width/2; x++) {
			frame->data[1][y * frame->linesize[1] + x] = 128 + y + i * 2;
			frame->data[2][y * frame->linesize[2] + x] = 64 + x + i * 5;
		}
	}
	frame->pts = i; //显示时间

	//发送未压缩帧到线程中压缩
	re = avcodec_send_frame(c, frame);
	if (re != 0) {
		break;
	}

	while (re >= 0) {	//返回多帧
		//接收压缩帧，一般前几次调用return 空(缓冲，立刻返回，编码未完成)
		//编码是在独立的线程中
		//每次调用会重写分配pkt中的空间
		re = avcodec_receive_packet(c, pkt);
		if (re == AVERROR(EAGAIN) || re == AVERROR_EOF) { break; }
		if(re < 0){ 
			char buf[1024]{ 0 };
			av_strerror(re, buf, sizeof(buf) - 1);
			std::cerr << "avcodec_receive_packet failed !" << buf << std::endl;
			break;
		}
		std::cout << pkt->size << " " << std::flush;
		av_packet_unref(pkt);
	}
}
```


# 视频H264(AVC)编码流程

![[Pasted image 20240813201157.png]]
## 宏块划分 Macro Block
H264/H265/H266 三种视频编码都是基于块进行的划分为更小单元的：将一帧视频（即一张图片）划分成不同的块，然后对每个块再分别进行编码处理。从H264到H265，再从H265到H266，块的划分精度越来越高。
** H264默认是使用 16X16 大小的区域作为一个宏块，也可以划分成 8X8 大小。**

## 帧内预测 I帧
- 有损压缩 类似jpeg
- 每个宏块可以进行9种模式的预测
- 原始图像与帧内预测后的图像相减得残差值
- 空间冗余
- 压缩率低和独立解码
具体如图 ![[Pasted image 20240813201957.png]]
**I帧**，也称为**关键帧**或帧**内编码帧**（Intra-coded Frame），是一个完整的图像帧，它独立于其他帧存在。I帧不依赖于其他帧的信息即可独立解码，类似于静态图像，可以视为视频序列中的一个参考点。
**` 注意 ：由于I帧包含了完整的图像信息，其压缩率相对较低，但在解码时最为简单，因为它不涉及对其他帧的依赖。`**

## 帧间预测和GOP P帧和B帧

- 时间冗余
- 相邻帧之间内容相似，有运动关系
- 根据前一帧，或者I帧运动矢量的补偿
- 多帧预测功能，可选5个参考帧，纠错
- 去块滤波器，水平和垂直块边缘锯齿
- GOP
- P帧 向前参考帧         压缩率高于I帧
- B帧 双向参考帧         压缩率更高

**P帧：预测帧（Predicted Frame)**
P帧，即前向预测编码帧（Predictive Frame），依赖于前面的I帧或P帧来生成。P帧存储的是与前一帧相比图像的变化量，因此它的压缩效果通常比I帧更好。在解码P帧时，需要先解码它所依赖的I帧或P帧，然后根据这些信息来重建当前帧的画面。P帧的引入有效减少了时间维度上的冗余，提高了视频的压缩效率。

**B帧：双向预测帧（Bidirectional Frame)**
B帧，或称为双向预测内插编码帧（Bidirectional Interpolated Prediction Frame），需要参考前后的I帧或P帧来生成。B帧利用前后帧的信息来预测当前帧的内容，从而实现更高的压缩比。由于B帧的解码需要前后帧的信息，它不能独立解码，**必须在解码序列中结合I帧和P帧来完成。**

>I帧、P帧和B帧之间的联系体现在视频编码和解码的过程中。I帧作为参考点，为P帧和B帧提供了必要的信息。P帧和B帧通过引用I帧来减少数据量，实现高效的视频压缩。在视频播放时，解码器会根据这些帧的顺序和相互关系来重建视频序列，确保视频的连续性和流畅性。
** **
 >解码过程通常从I帧开始，然后依次解码P帧和B帧。I帧由于是独立帧，可以首先被解码。随后，解码器利用I帧的信息来解码P帧，最后结合I帧和P帧的信息来解码B帧。这种解码顺序确保了视频帧能够按照正确的时间顺序显示。

**GOP(Group of Pictures)**
顾名思义，就是一组图片，在实际操作中，就是一组完整的视频帧，怎么叫做完整的视频帧？也就是说一个GOP拿出来，必须能够完整的播放、显示。
例如： 
		I PPPP ... PPBP I ....
**`以I帧开头到下一个I帧之前，称之为GOP`**


## 变换__量化和熵编码__变长和算数

** DCT离散余弦变换 **
- 去除空间信号的相关性
- 离散余弦变换后变成一组系数，每个系数是标准奇函数的加权值，可重建残差值

** 量化 **
- 在数字信号处理是连续的模拟信号到数字信号
- 多对一的过程，
- 量化控制码率 
- 对变换系数做量化
-  反量化就是一对一映射
- QP量化参数
- FQ = round(y/Qstep)
- 量化参数 qmax qmin

** 熵编码 **
- 熵在热力学中，是表示分子状态混乱程度的物理量
- 去除信息表达中的冗余
- 无损压缩
- 变长编码，哈夫曼编码，指数哥伦布编码，CAVLC
	-   大概率符号分配短的码，小概率分配长的
- 算数编码 CABAC
	-   算数编码的本质是为整个输入序列分配一个码字
	-   CABAC 二进制算术编码
	-  上下文建模=》二值化 =》概率估计 =》编码引擎
	-  编码器在开始时将“当前间隔”设置为(L,H]
	-  编码器将“当前间隔”分为子间隔，每一个事件一个。

# perset预设编码器
`preset` 是 FFmpeg 中用于指定视频编码预设配置的参数，它允许用户根据需要选择不同的编码速度和输出质量之间的平衡。预设配置是一组预定义的编码参数，方便用户在不同场景下快速应用
>`preset` 的参数主要调节编码速度和质量的平衡，有 `ultrafast`（转码速度最快，视频质量可能较低）、`superfast`、`veryfast`、`faster`、`fast`、`medium`（默认设置）、`slow`、`slower`、`veryslow` 和 `placebo` 这 10 个选项，从快到慢排列.选择的 `preset` 会影响编码器使用的计算资源和最终输出视频的质量。例如，`ultrafast` 预设会使用较少的计算资源，但输出视频的质量可能不如 `veryslow` 预设，后者会使用更多的计算资源以提高视频质量.

调用av_opt_set来预设编码器参数(**在打开编码上下文之前**)
```cpp
//预设编码器参数
c->max_b_frames = 0;		//B帧设为0，降低延时，增大空间。
int opt_re = av_opt_set(c->priv_data,"preset","ultrafast",0);	//最快速度
if (opt_re != 0) {
	std::cout << "preset failed !" << std::endl;
}
av_opt_set(c->priv_data, "tune", "zerolatency", 0); //0延时 h265不支持b_frame

//4 打开编码上下文
/*.......*/
```
使用**BitrateViewer**查看，生成之后的码率更高。
这意味着编码器会使用更快的算法来减少编码时间，但这些算法可能不会进行足够的分析来找到最佳的压缩方式，从而导致生成的文件比使用其他预设选项时更大。
![[Pasted image 20240814130344.png]]

## 比特率控制
### **ABR平均比特率**
它以固定的平均比特率来编码视频。这意味着编码器会在整个视频长度上**平均分配比特率**，但允许在不同场景下根据需要动态调整比特率。
- **优点**：
    - 可以根据视频内容的复杂性动态调整比特率，从而在保持整体质量的同时优化文件大小。
    - 适合于视频内容变化较大或有大量动作的场景。
- **缺点**：
    - 可能导致视频质量在不同场景下波动，特别是在比特率较低时。
```cpp
/***************************
*  ABR 平均比特率
*/
int br = 400000;  //400kb
c->bit_rate = br;
```
![[Pasted image 20240814150913.png]]
![[Pasted image 20240814150945.png]]
可见，将平均码率控制在了45kb左右，**第一帧为I帧**。

### **CQP恒定质量**
它使用固定的质量参数来编码视频。在这种模式下，编码器会尝试保持视频的整体质量一致，而不管视频内容的复杂性如何。CQP 通常与 CRF（Constant Rate Factor）或类似的量化参数一起使用，这些参数指定了编码后视频的质量级别。
- **优点**：由于质量保持一致，用户在观看视频时不会遇到质量波动。
- **缺点**：如果视频内容复杂，可能会导致某些场景的码率过高，从而增加文件大小。

>`量化参数（Quantization Parameter，简称QP）是视频编码中一个重要的概念，它控制着编码过程中量化步长的大小，从而影响视频的压缩效率和质量。量化过程是将变换后的系数映射到离散值的过程，是造成图像失真的根本原因。量化步长Qstep随着QP的增加而增加，每当QP值增加6，Qstep便增加一倍，量化最精细时QP取最小值0，而量化最粗糙时QP取最大值51.
>`QP 值小，更多细节就会保留；QP 值越大，就会丢失更多细节，视频画面的清晰度就会越低。也就是说，QP 和比特率成反比的关系，而且随着视频源复杂度的提高，这种反比关系会更明显。`

```cpp
/***************************
*  CQP 恒定质量 H264中的QP范围从0到51
*  x264默认23 效果较好18
*  x265默认28 效果较好25
*/
av_opt_set_int(c->priv_data, "qp", 51,0);
```
![[Pasted image 20240814145448.png]]
可以看到，生成后的视频平均码率在74kb左右
![[Pasted image 20240814145705.png ]]
且视频中马赛克明显.
****
若更改其中的**QP量化因子**。
```cpp
av_opt_set_int(c->priv_data, "qp", 18, 0);
```
![[Pasted image 20240814145938.png]]
可见，平均码率提高为183kb左右，且视频质量更加好。
根据日志也可得到**I帧 QP 值较小，P帧和B帧的 QP 值较大**
![[Pasted image 20240814145952.png]]

### **恒定比特率(CBR)**
CBR的兼容性和稳定性较高，但是存在资源浪费：
**比如在简单的场景中，CBR会使用过多的比特，导致存储空间和带宽的浪费。
而在复杂的场景中，CBR会导致视频质量波动，因为编码器必须在保持恒定码率的同时压缩视频，可能会牺牲一些细节。**
```cpp
/***************************
*  恒定比特率(CBR)由于MP4不支持NAL填充，因此输出文件必须为（MPEG-2 TS）
*/
int br = 400000;  //400kb
c->rc_min_rate = br;
c->rc_max_rate = br;
c->rc_buffer_size = br;
c->bit_rate = br;
av_opt_set(c->priv_data, "nal-hrd", "cbr", 0);
```
![[Pasted image 20240814172335.png]]
根据日志可得**每帧的大小比较固定**
![[Pasted image 20240814172443.png]]
平均码率为400kb左右，比特率非常的稳定，
### **恒定速率因子(CRF)和约束编码(VBV)**

```cpp
/***************************
*  恒定速率因子(CRF)
*/
av_opt_set_int(c->priv_data, "crf", 23, 0);

/***************************
*  约束编码(VBV) Constrained Encoding (VBV)
*/
av_opt_set_int(c->priv_data, "crf", 23, 0);
c->rc_max_rate = br;
c->rc_buffer_size = br * 2;
```
``c->rc_buffer_size` 设置了编码器的码率控制缓冲区的大小，这里设置为 `br * 2`。缓冲区用于平滑码率的波动，确保在视频内容变化时，输出码率不会超过 `rc_max_rate`。`
>CRF提供了一个质量控制机制，允许编码器在保持一定质量水平的同时动态调整码率。通过设置CRF值，编码器可以在不同场景下分配适当的比特率，以保持视频质量.
而VBV通过设置最大码率（`rc_max_rate`）和缓冲区大小（`rc_buffer_size`），可以平滑视频的码率波动。当场景复杂度增加时，编码器可以暂时**增加码率**，但**不会超过设定的最大码率，从而避免突然的码率峰值**。
结合使用CRF和VBV可以提高编码的适应性。编码器可以根据视频内容的复杂性以及网络条件或播放环境的要求，灵活调整编码策略。

## H264码流中NALU sps pps IDR帧的理解

**H.264**原始码流**NALU**，分两层
- **VCL** : 包括核心压缩引擎和块，宏块和片的语法级别定义。
    - 包含所有核心的压缩的数据，即最终编码生成的数据（**宏块划分，预测，量化，熵编码**）
- **NAL**：负责将VCL产生的比特字符串适配到各种各样的网络和多元环境中
- **VCL**数据封装进NAL单元 （**NALU**）

**NAL Header**
- 由三部分组成，forbidden_bit(1bit), nal_reference_bit(2bit) (优先级),nal_unit_type(5bits)（类型）
- 包之间间隔 0001 或 001

**NALU的类型**
- 0 ：未规定
- 1 : 非IDR图像中不采用数据划分的片段
- 5：IDR图像的片段
- 6：补充增强信息(SEI0)
- 7：序列参数集/SPS
- 8：图像参数集/PPS
- SPS和PPS都是特殊的NALU。一个MP4文件只有一个SPS，但是有很多PPS,SPS必须在所有NALU的最开头

 **SPS（Sequence Parameter Set）**：
- SPS包含了整个视频序列的全局参数，这些参数在整个视频序列中保持不变。
- 它定义了视频的分辨率、帧率、色彩空间、编码方式等基本参数。
- SPS还包含了一些编码工具的配置信息，如量化参数、去块滤波器等。

 **PPS（Picture Parameter Set）**：
- PPS包含了图像级别的参数，这些参数可以针对每个图像进行调整。
- 它定义了图像的编码模式，如帧内预测、运动估计和补偿等。
- PPS还包含了一些图像编码的控制参数，例如宏块（MB）的分割方式、去块滤波器的开关状态等。

`解码器在解码视频流之前，首先需要解析SPS和PPS，以获取必要的编码参数，从而正确地解码后续的视频帧`

**IDR(Instantaneous Decoding Refresh)**
IDR一定是I帧，但I帧不一定是IDR。一旦出现IDR，就表示清除前面的序列，并且立刻渲染当前的IDR帧。