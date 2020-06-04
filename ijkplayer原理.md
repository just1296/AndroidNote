# ijkplayer原理

## 1、目录结构
> config：编译ffmpeg使用的配置文件
> 
> extra：存放ijkplayer依赖的文件，ffmpeg、libyuv等

> ijkmedia：核心部分
>> ijkplayer：播放器数据下载及解码相关
>> 
>> ijksdl：音视频数据渲染相关

> tools：初始化项目工程脚本
> 
> android/ios：android和ios平台上的上层接口封装、平台相关方法

## 2、初始化流程
初始化ijkplayer从Java类`IjkMediaPlayer`的构造函数开始，内部调用初始化播放器方法`initPlayer()`

```java
private void initPlayer(IjkLibLoader libLoader) {
    loadLibrariesOnce(libLoader);
    ...

    /*
     * Native setup requires a weak reference to our object. It's easier to
     * create it here than in C++.
     */
    native_setup(new WeakReference<IjkMediaPlayer>(this));
}

private native void native_setup(Object IjkMediaPlayer_this);
```
首先调用`loadLibrariesOnce()`方法初始化so文件加载器，加载ijkffmpeg、ijksdl、ijkplayer这三个so文件。接着调用`native_setup()`方法初始化播放器，将IjkMediaPlayer对象通过弱引用传递给native层。

`native_setup()`方法在native层的注册在ijkplayer_jni.c中进行：

```
static JNINativeMethod g_methods[] = {
...
	{ "native_setup", "(Ljava/lang/Object;)V", (void *) IjkMediaPlayer_native_setup },
...
};

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
    JNIEnv* env = NULL;

    g_jvm = vm;
    if ((*vm)->GetEnv(vm, (void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        return -1;
    }
    assert(env != NULL);

    pthread_mutex_init(&g_clazz.mutex, NULL );

    // FindClass returns LocalReference
    IJK_FIND_JAVA_CLASS(env, g_clazz.clazz, JNI_CLASS_IJKPLAYER);
    (*env)->RegisterNatives(env, g_clazz.clazz, g_methods, NELEM(g_methods) );

    ijkmp_global_init();
    ijkmp_global_set_inject_callback(inject_callback);

    FFmpegApi_global_init(env);

    return JNI_VERSION_1_4;
}

```
首先将`native_setup()`方法放入方法数组`g_method`中，并与native方法`IjkMediaPlayer_native_setup()`关联，然后在`JNI_OnLoad`中注册本地方法。`IjkMediaPlayer_native_setup()`方法如下：

```
static void
IjkMediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    MPTRACE("%s\n", __func__);
    IjkMediaPlayer *mp = ijkmp_android_create(message_loop);
    JNI_CHECK_GOTO(mp, env, "java/lang/OutOfMemoryError", "mpjni: native_setup: ijkmp_create() failed", LABEL_RETURN);

    jni_set_media_player(env, thiz, mp);
    ijkmp_set_weak_thiz(mp, (*env)->NewGlobalRef(env, weak_this));
    ijkmp_set_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_set_ijkio_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_android_set_mediacodec_select_callback(mp, mediacodec_select_callback, ijkmp_get_weak_thiz(mp));

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```
调用`ijkmp_android_create()`方法：

```
IjkMediaPlayer *ijkmp_android_create(int(*msg_loop)(void*))
{
	// 1、创建IjkMediaPlayer对象
    IjkMediaPlayer *mp = ijkmp_create(msg_loop);
    if (!mp)
        goto fail;

	// 2、创建图像渲染对象SDL_Vout
    mp->ffplayer->vout = SDL_VoutAndroid_CreateForAndroidSurface();
    if (!mp->ffplayer->vout)
        goto fail;

	// 3、创建平台相关的ff_pipeline，包括视频解码以及音频输出部分
    mp->ffplayer->pipeline = ffpipeline_create_from_android(mp->ffplayer);
    if (!mp->ffplayer->pipeline)
        goto fail;

    ffpipeline_set_vout(mp->ffplayer->pipeline, mp->ffplayer->vout);

    return mp;

fail:
    ijkmp_dec_ref_p(&mp);
    return NULL;
}
```
该方法主要完成三件事：

> 1、创建IjkMediaPlayer对象
> 
> 2、创建图像渲染对象SDL_Vout
> 
> 3、创建平台相关的ff_pipeline，包括视频解码以及音频输出部分

## 3、prepare流程
prepare从外界调用IMediaPlayer接口`prepareAsync()`方法开始，IjkMediaPlayer实现IMediaPlayer接口，调用native方法`_prepareAsync()`，该方法对应jni函数`IjkMediaPlayer_prepareAsync`

```
static JNINativeMethod g_methods[] = {
	...
	{ "_prepareAsync", "()V", (void *) IjkMediaPlayer_prepareAsync },
	...
}

static void
IjkMediaPlayer_prepareAsync(JNIEnv *env, jobject thiz)
{
    ...
    int retval = 0;
    IjkMediaPlayer *mp = jni_get_media_player(env, thiz);
    ...
    retval = ijkmp_prepare_async(mp);
	...
}
```

```
int ijkmp_prepare_async(IjkMediaPlayer *mp)
{
    ...
    pthread_mutex_lock(&mp->mutex);
    int retval = ijkmp_prepare_async_l(mp);
    pthread_mutex_unlock(&mp->mutex);
    ...
    return retval;
}
```

```
static int ijkmp_prepare_async_l(IjkMediaPlayer *mp)
{
	...
	int retval = ffp_prepare_async_l(mp->ffplayer, mp->data_source);
	...
}
```

```
int ffp_prepare_async_l(FFPlayer *ffp, const char *file_name)
{
	...
	av_opt_set_dict(ffp, &ffp->player_opts);
    if (!ffp->aout) {
        ffp->aout = ffpipeline_open_audio_output(ffp->pipeline, ffp);
        if (!ffp->aout)
            return -1;
    }
    
	VideoState *is = stream_open(ffp, file_name, NULL);
	...
}
```
可以看到，prepare阶段最后调用到ffplay.c中的`ffp_prepare_async_l()`方法，该方法会设置player选项，打开audio output，并且调用`stream_open()`方法。

```
static VideoState *stream_open(FFPlayer *ffp, const char *filename, AVInputFormat *iformat)
{
	...
	/* start video display */
    if (frame_queue_init(&is->pictq, &is->videoq, ffp->pictq_size, 1) < 0)
        goto fail;
    if (frame_queue_init(&is->subpq, &is->subtitleq, SUBPICTURE_QUEUE_SIZE, 0) < 0)
        goto fail;
    if (frame_queue_init(&is->sampq, &is->audioq, SAMPLE_QUEUE_SIZE, 1) < 0)
        goto fail;

    if (packet_queue_init(&is->videoq) < 0 ||
        packet_queue_init(&is->audioq) < 0 ||
        packet_queue_init(&is->subtitleq) < 0)
        goto fail;
    ...
    is->video_refresh_tid = SDL_CreateThreadEx(&is->_video_refresh_tid, video_refresh_thread, ffp, "ff_vout");
    ...
    is->read_tid = SDL_CreateThreadEx(&is->_read_tid, read_thread, ffp, "ff_read");
    ...
}
```
stream_open主要做了以下几件事：

> 1、创建存放video/audio/subtitle解码前数据的videoq/audioq/subtitleq
> 
> 2、创建存放video/audio/subtitle解码后数据的videoq/audioq/subtitleq
> 
> 3、创建视频渲染线程`video_refresh_thread`
> 
> 4、创建读数据线程`read_thread`

## 4、数据读取
在prepare阶段，创建读数据线程时，指定了读数据操作的方法指针是ffmpeg内部的ffplay.c的`read_thread()`。该方法负责接收从网络/本地读取的数据，根据视频的封装格式，完成解复用动作。

```
/* this thread gets the stream from the disk or the network */
static int read_thread(void *arg)
{
	VideoState *is = arg;
	...
	// 1、创建上下文结构体，表示输入上下文
	ic = avformat_alloc_context();
	...
	// 2、设置中断函数，如果出错或者退出
	ic->interrupt_callback.callback = decode_interrupt_cb;
    ic->interrupt_callback.opaque = is;
    ...
    // 3、打开文件，探测协议类型，如果是网络文件则创建网络链接
    err = avformat_open_input(&ic, is->filename, is->iformat, &ffp->format_opts);
    ...
    // 4、探测媒体类型，得到当前文件的封装格式，音视频编码参数新
    err = avformat_find_stream_info(ic, opts);
    ...
    // 5、打开视频、音频、字幕解码器，创建相应的解码线程
    if (st_index[AVMEDIA_TYPE_AUDIO] >= 0) {
        stream_component_open(is, st_index[AVMEDIA_TYPE_AUDIO]);
    }
    ret = -1;
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
        ret = stream_component_open(is, st_index[AVMEDIA_TYPE_VIDEO]);
    }
    if (st_index[AVMEDIA_TYPE_SUBTITLE] >= 0) {
        stream_component_open(is, st_index[AVMEDIA_TYPE_SUBTITLE]);
    }
    ...
    // 开启循环，不断获取待播放的数据
    for (;;) {
    	...
	    // 6、读取媒体数据，得到音视频分离的解码前数据
	    ret = av_read_frame(ic, pkt);
    	...
    	// 7、将音视频数据分别送入相应的queue中
    	if (pkt->stream_index == is->audio_stream && pkt_in_play_range) {
            packet_queue_put(&is->audioq, pkt);
        } else if (pkt->stream_index == is->video_stream && pkt_in_play_range
                   && !(is->video_st->disposition & AV_DISPOSITION_ATTACHED_PIC)) {
            packet_queue_put(&is->videoq, pkt);
        } else if (pkt->stream_index == is->subtitle_stream && pkt_in_play_range) {
            packet_queue_put(&is->subtitleq, pkt);
        } else {
            av_packet_unref(pkt);
        }
    }
    ...
}
```

## 5、音视频解码
ijkplayer在视频解码上支持软解和硬解两种方式，可以在起播前配置优先使用的解码方式，播放过程中不可切换。Android平台硬解使用MediaCodec，iOS平台硬解使用VideoToolbox。ijkplayer音频解码只支持软解，暂不支持硬解。

### 5.1 视频解码方式选择
ijkplayer的解码器对应ff_ffplay.c中的`stream_component_open()`方法

```
/* open a given stream. Return 0 if OK */
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
	AVCodecContext *avctx;
	AVCodec *codec = NULL;
	...
	// 创建解码器上下文
	avctx = avcodec_alloc_context3(NULL);
	...
	codec = avcodec_find_decoder(avctx->codec_id);
	...
	switch (avctx->codec_type) {
	// 音频
	case AVMEDIA_TYPE_AUDIO:
		...
		decoder_init(&is->auddec, avctx, &is->audioq, is->continue_read_thread);
		...
		// 打开音频解码器，音频解码线程对应的回调方法：audio_thread
		if ((ret = decoder_start(&is->auddec, audio_thread, ffp, "ff_audio_dec")) < 0)
            goto out;
        ...
        break;
	...
	// 视频
	case AVMEDIA_TYPE_VIDEO:
		...
		decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
		// 打开视频解码器
    	ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
    	if (!ffp->node_vdec)
            goto fail;
    	...
    	// 视频解码线程对应的回调方法：video_thread
    	if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
            goto out;
}
```
在调用`ffpipeline_open_video_decoder()`打开解码器时，会根据pipeline打开对应平台的解码器，Android平台对应ffpipeline_android.c中的`func_open_video_decoder()`方法：

```
static IJKFF_Pipenode *func_open_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    IJKFF_Pipenode        *node = NULL;

    if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2)
    	// 硬解
        node = ffpipenode_create_video_decoder_from_android_mediacodec(ffp, pipeline, opaque->weak_vout);
    if (!node) {
    	// 软解
        node = ffpipenode_create_video_decoder_from_ffplay(ffp);
    }
    return node;
}
```
如果配置了mediacodec\_all\_videos、mediacodec\_avc、mediacodec\_hevc或mediacodec\_mpeg2，会优先尝试打开硬件解码器MediaCodec。如果硬解码器打开失败，则切换到软解。

### 5.2 视频解码
视频解码线程对应回调方法是ff\_ffplay.c中的`video_thread()`方法：

```
static int video_thread(void *arg)
{
    FFPlayer *ffp = (FFPlayer *)arg;
    int       ret = 0;

    if (ffp->node_vdec) {
        ret = ffpipenode_run_sync(ffp->node_vdec);
    }
    return ret;
}
```

```
int ffpipenode_run_sync(IJKFF_Pipenode *node)
{
    return node->func_run_sync(node);
}
```
`func_run_sync()`方法取决于播放前配置的软硬解.

- 如果是软解，调用ffpipenode\_ffplay\_vdec.c中的`ffplay_video_thread()`
	
	```
	static int ffplay_video_thread(void *arg)
	{
		FFPlayer *ffp = arg;
		AVFrame *frame = av_frame_alloc();
		...
		for (;;) {
			ret = get_video_frame(ffp, frame);
			...
			ret = queue_picture(ffp, frame, pts, duration, frame->pkt_pos, is->viddec.pkt_serial);
			...
		}
	}
	```
	`get_video_frame()`调用了`decoder_decode_frame()`方法，从解码前的video queue中取出一帧数据，送入decoder进行解码，解码后的数据在`ffplay_video_thread`中送入pictq。
	
	```
	static int decoder_decode_frame(FFPlayer *ffp, Decoder *d, AVFrame *frame, AVSubtitle *sub) {
	    int ret = AVERROR(EAGAIN);
	
	    for (;;) {
	        AVPacket pkt;
	        do {
	        	switch (d->avctx->codec_type) {
	                case AVMEDIA_TYPE_VIDEO:
	                	ret = avcodec_receive_frame(d->avctx, frame);
	                	...
	                	break;
	        } while (ret != AVERROR(EAGAIN));
	        ...
	    }
	}
	```
- 如果是硬解，在ffpipenode\_android\_mediacodec\_vdec.c内部判断是否设置了mediacodec\_sync，如果设置了就调用`func_run_sync_loop()`，否则调用`func_run_sync()`。如果硬解码器不存在，这两个方法内部调用了ff\_ffplay.c中`ffp_video_thread`进行软解。

### 5.3 音频解码
音频解码线程对应回调方法是ff\_ffplay.c中的`audio_thread()`方法：

```
static int audio_thread(void *arg)
{
	FFPlayer *ffp = arg;
	AVFrame *frame = av_frame_alloc();
	...
	do {
		if ((got_frame = decoder_decode_frame(ffp, &is->auddec, frame, NULL)) < 0)
            goto the_end;
       ...
	} while (ret >= 0 || ret == AVERROR(EAGAIN) || ret == AVERROR_EOF);
	...
}
```
和视频解码一样，`audio_thread()`内部也调用了`decoder_decode_frame()`方法

```
static int decoder_decode_frame(FFPlayer *ffp, Decoder *d, AVFrame *frame, AVSubtitle *sub) {
    int ret = AVERROR(EAGAIN);
	
    for (;;) {
        AVPacket pkt;
        do {
        	switch (d->avctx->codec_type) {
                case AVMEDIA_TYPE_AUDIO:
                    ret = avcodec_receive_frame(d->avctx, frame);
                	...
                	break;
        } while (ret != AVERROR(EAGAIN));
        ...
    }
}
```

## 6、音视频渲染
### 6.1 视频渲染
prepare流程部分提到，会创建视频渲染线程，对应回调方法是`video_refresh_thread()`，该方法内部最后调用`video_image_display2()`方法：

```
static void video_image_display2(FFPlayer *ffp)
{
    VideoState *is = ffp->is;
    Frame *vp;
    Frame *sp = NULL;
    
    // 从pictq中读取当前需要显示视频帧
    vp = frame_queue_peek_last(&is->pictq);
    ...
    // 绘制视频画面
    SDL_VoutDisplayYUVOverlay(ffp->vout, vp->bmp);
    ...
}
```
SDL\_VoutDisplayYUVOverlay方法内部调用`display_overlay()`方法进行绘制视频画面，该方法在Android平台对应ijksdl\_vout\_android\_nativewindow.c下的`fun_display_overlay()`方法。内部通过OpenGL、OpenGL ES和两者相结合的方式实现。

```
int SDL_VoutDisplayYUVOverlay(SDL_Vout *vout, SDL_VoutOverlay *overlay)
{
    if (vout && overlay && vout->display_overlay)
        return vout->display_overlay(vout, overlay);
    return -1;
}
```

### 6.2 音频输出
ijkplayer中Android平台使用OpenSL ES或AudioTrack输出音频，iOS平台使用AudioQueue输出音频。

在prepare流程部分提到，`ffp_prepare_async_l()`方法中会创建audio output：

```
if (!ffp->aout) {
    ffp->aout = ffpipeline_open_audio_output(ffp->pipeline, ffp);
    if (!ffp->aout)
        return -1;
}
```
该方法内部调用pipeline的`fun_open_audio_output()`方法，在ffpipeline\_android.c中pipeline->func\_open\_audio\_output被指向`func_open_audio_output()`方法：

```
static SDL_Aout *func_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    SDL_Aout *aout = NULL;
    if (ffp->opensles) {
    	// 使用OpenSL ES
        aout = SDL_AoutAndroid_CreateForOpenSLES();
    } else {
    	// 使用AudioTrack
        aout = SDL_AoutAndroid_CreateForAudioTrack();
    }
    if (aout)
        SDL_AoutSetStereoVolume(aout, pipeline->opaque->left_volume, pipeline->opaque->right_volume);
    return aout;
}
```

如果待播放的文件含有音频，会调用ffplay.c中的`stream_component_open()`方法打开解码器，该方法中调用`audio_open`打开audio output设备：

```
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
	...
	if ((ret = audio_open(ffp, channel_layout, nb_channels, sample_rate, &is->audio_tgt)) < 0)
        goto fail;
    ...
}
```

```
static int audio_open(FFPlayer *opaque, int64_t wanted_channel_layout, int wanted_nb_channels, int wanted_sample_rate, struct AudioParams *audio_hw_params)
{
	FFPlayer *ffp = opaque;
    VideoState *is = ffp->is;
    SDL_AudioSpec wanted_spec, spec;
    ...
    wanted_nb_channels = av_get_channel_layout_nb_channels(wanted_channel_layout);
    wanted_spec.channels = wanted_nb_channels;
    wanted_spec.freq = wanted_sample_rate;
    wanted_spec.format = AUDIO_S16SYS;
    wanted_spec.silence = 0;
    wanted_spec.samples = FFMAX(SDL_AUDIO_MIN_BUFFER_SIZE, 2 << av_log2(wanted_spec.freq / SDL_AoutGetAudioPerSecondCallBacks(ffp->aout)));
    // 读取音频数据的缓冲回调
    wanted_spec.callback = sdl_audio_callback;
    wanted_spec.userdata = opaque;
    while (SDL_AoutOpenAudio(ffp->aout, &wanted_spec, &spec) < 0) {
        ...
    }
    ...
    return spec.size;
}
```
`audio_open()`方法中使用`SDL_AoutOpenAudio()`方法打开音频文件：

```
int SDL_AoutOpenAudio(SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained)
{
    if (aout && desired && aout->open_audio)
        return aout->open_audio(aout, desired, obtained);

    return -1;
}
```
在ijksdl\_aout\_android\_audiotrack.c和ijksdl\_aout\_android\_opensles.c中，aout->open\_audio都被指向了`aout_open_audio()`方法。该方法内部调用了`aout_open_audio_n()`方法：

```
static int aout_open_audio_n(JNIEnv *env, SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained)
{
	SDL_Aout_Opaque *opaque = aout->opaque;
	...
	opaque->audio_tid = SDL_CreateThreadEx(&opaque->_audio_tid, aout_thread, aout, "ff_aout_android");
	...
}
```
通过创建输出音频线程，并将线程指向`aout_thread()`方法，该方法内部又调用了`aout_thread_n()`方法：

```
static int aout_thread_n(JNIEnv *env, SDL_Aout *aout)
{
    SDL_Aout_Opaque *opaque = aout->opaque;
    SDL_Android_AudioTrack *atrack = opaque->atrack;
    SDL_AudioCallback audio_cblk = opaque->spec.callback;
    void *userdata = opaque->spec.userdata;
    uint8_t *buffer = opaque->buffer;
    int copy_size = 256;
    
    ...
    while (!opaque->abort_request) {
    	...
    	audio_cblk(userdata, buffer, copy_size);
    	...
	    if (opaque->need_flush) {
	        opaque->need_flush = 0;
	        SDL_Android_AudioTrack_flush(env, atrack);
	    } else {
	        int written = SDL_Android_AudioTrack_write(env, atrack, buffer, copy_size);
	        if (written != copy_size) {
	            ALOGW("AudioTrack: not all data copied %d/%d", (int)written, (int)copy_size);
	        }
	    }
    }
    ...
}
```
拿到设置的callback的引用，不断调用callback方法获取pcm数据，这里的callback方法是ff_ffplay.c中的`sdl_audio_callback()`方法。

### 6.3 音视频同步
通常音视频同步的解决方案是选择一个参考时钟，播放时读取音视频帧上的时间戳，同时参考当前时钟参考时钟上的时间来安排播放。

如果音视频帧的播放时间大于当前参考时钟上的时间，则不急于播放该帧，直到参考时钟达到该帧的时间戳；如果音视频帧的时间戳小于当前参考时钟上的时间，则需要"尽快"播放该帧或丢弃，以便播放进度追上参考时钟。

ijkplayer在默认情况下使用音频作为参考时钟源，处理同步的过程主要在视频渲染`video_refresh_thread`的线程中：

```
static int video_refresh_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    double remaining_time = 0.0;
    while (!is->abort_request) {
        if (remaining_time > 0.0)
            av_usleep((int)(int64_t)(remaining_time * 1000000.0));
        remaining_time = REFRESH_RATE;
        if (is->show_mode != SHOW_MODE_NONE && (!is->paused || is->force_refresh))
        	// 刷新视频帧，计算remaining_time
            video_refresh(ffp, &remaining_time);
    }
    return 0;
}
```
核心处理逻辑在`video_refresh()`方法中：

```
/* called to display each frame */
static void video_refresh(FFPlayer *opaque, double *remaining_time)
{
	VideoState *is = ffp->is;
	...
	if (is->video_st) {
retry:
		...
		 /* dequeue the picture */
        lastvp = frame_queue_peek_last(&is->pictq);
        vp = frame_queue_peek(&is->pictq);
		...
		/* compute nominal last_duration */
        last_duration = vp_duration(is, lastvp, vp);
        delay = compute_target_delay(ffp, last_duration, is);
        ...
	}
}
```
对于视频帧，从帧队列中取出lastvp(上一个视频帧)和vp(当前视频帧)，last_duration是根据lastvp和vp计算出上一帧的时长，通过`compute_target_delay()`计算出当前帧需要等待的时长。

```
static double compute_target_delay(FFPlayer *ffp, double delay, VideoState *is)
{
    double sync_threshold, diff = 0;

    /* update delay to follow master synchronisation source */
    if (get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER) {
        /* if video is slave, we try to correct big delays by
           duplicating or deleting a frame */
        diff = get_clock(&is->vidclk) - get_master_clock(is);

        /* skip or repeat frame. We take into account the
           delay to compute the threshold. I still don't know
           if it is the best guess */
        sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
        /* -- by bbcallen: replace is->max_frame_duration with AV_NOSYNC_THRESHOLD */
        if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD) {
            if (diff <= -sync_threshold)
                delay = FFMAX(0, delay + diff);
            else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD)
                delay = delay + diff;
            else if (diff >= sync_threshold)
                delay = 2 * delay;
        }
    }
    ...
    return delay;
}
```
在`compute_target_delay()`方法中，如果当前主时钟源不是video类型，就计算当前视频时钟与主时钟的差值：

- 如果当前视频帧落后于主时钟源，则需要减小下一帧画面的等待时间。
- 如果视频帧超前，并且该帧的显示时间大于显示更新门槛，则显示下一帧的时间为上一帧的显示时间加上超前的时间差。
- 如果视频帧超前，并且上一帧的显示时间小于显示更新门槛，则采取加倍延时的策略。

