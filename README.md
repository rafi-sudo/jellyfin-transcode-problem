# jellyfin-transcode-problem
# JELLYFIN TRANSCODE PROBLEM

# A. DESCRIBE
Jellyfin membutuhkan hardware atau software decoding untuk melakukan transcode. Server saya pakai HG680P dengan Armbian Ubuntu 18 dan prosesor Amlogic S905X, ada beberapa kemungkinan kenapa transcode tidak jalan:
1. Amlogic S905X punya hardware decoding (GPU)Mali-450 MP3, tapi driver untuk Armbian mungkin tidak tersedia atau tidak diaktifkan.
2. Coba cek apakah VAAPI atau V4L2 tersedia
   jalankan ```vainfo``` di cli atau ```ls /dev/video*```
3. Kalau tidak ada output atau error, berarti hardware acceleration belum aktif atau driver tidak terinstall
   
Jellyfin butuh FFmpeg yang dikompilasi dengan dukungan hardware encoding. Cek dengan:
<pre>
<code id="code-block">
ffmpeg -hwaccels
</code>
</pre>
jika ffmpeg tidak memberi dukungan hardware encoding berarti transcode hanya bisa berjalan di CPU (software transcode), yang bisa terlalu berat untuk Amlogic S905X.
# B. ANSWER
<pre>
<code id="code-block">
ffmpeg -hwaccels
ffmpeg version 3.4.11-0ubuntu0.1 Copyright (c) 2000-2022 the FFmpeg developers
  built with gcc 7 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04)
  configuration: --prefix=/usr --extra-version=0ubuntu0.1 --toolchain=hardened --libdir=/usr/lib/aarch64-linux-gnu --incdir=/usr/include/aarch64-linux-gnu --enable-gpl --disable-stripping --enable-avresample --enable-avisynth --enable-gnutls --enable-ladspa --enable-libass --enable-libbluray --enable-libbs2b --enable-libcaca --enable-libcdio --enable-libflite --enable-libfontconfig --enable-libfreetype --enable-libfribidi --enable-libgme --enable-libgsm --enable-libmp3lame --enable-libmysofa --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libpulse --enable-librubberband --enable-librsvg --enable-libshine --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libssh --enable-libtheora --enable-libtwolame --enable-libvorbis --enable-libvpx --enable-libwavpack --enable-libwebp --enable-libx265 --enable-libxml2 --enable-libxvid --enable-libzmq --enable-libzvbi --enable-omx --enable-openal --enable-opengl --enable-sdl2 --enable-libdc1394 --enable-libdrm --enable-libiec61883 --enable-chromaprint --enable-frei0r --enable-libopencv --enable-libx264 --enable-shared
  libavutil      55. 78.100 / 55. 78.100
  libavcodec     57.107.100 / 57.107.100
  libavformat    57. 83.100 / 57. 83.100
  libavdevice    57. 10.100 / 57. 10.100
  libavfilter     6.107.100 /  6.107.100
  libavresample   3.  7.  0 /  3.  7.  0
  libswscale      4.  8.100 /  4.  8.100
  libswresample   2.  9.100 /  2.  9.100
  libpostproc    54.  7.100 / 54.  7.100
Hardware acceleration methods:
vdpau
vaapi
</code>
</pre>
Dari output ```ffmpeg -hwaccels```, perangkatmu mendukung dua metode akselerasi hardware:
1. VDPAU (Video Decode and Presentation API for Unix) → Biasanya untuk GPU NVIDIA atau perangkat tertentu.
2. VAAPI (Video Acceleration API) → Digunakan untuk GPU Intel dan beberapa GPU AMD/ARM.
Karena server HG680P-ku menggunakan Amlogic S905X, umumnya dukungan akselerasi hardware lebih ke V4L2 (Video4Linux2) atau libmali untuk GPU, bukan VDPAU/VAAPI.
Pastikan VAAPI dapat digunakan Jalankan perintah ini untuk melihat daftar perangkat VAAPI:
   
<pre>
<code id="code-block">
vainfo
</code>
</pre>
Jika vainfo belum terinstal, install dulu: ```sudo apt install vainfo -y```
# ANSWER : Sudah install vainfo keluar hanya kosong, sepertinya VAAPI tidak support karena driver tidak ada, saya pilih software akselerasi pakai VAAPI "tidak jalan transcode nya dengan notif source error" pakai V4L2 sama aja keluar " source error"
Iya dikarenakan HG680P menggunakan Amlogic S905X,  yang pada umumnya dukungan akselerasi hardware lebih ke V4L2 (Video4Linux2) atau libmali untuk GPU, bukan VDPAU/VAAPI jadi default driver terinstall dengan V4L2, tetapi saat pakai V4L2 kenapa ga bisa, saya curiga dilihat dari log ```ffmpeg -hwaccels``` hardware acceleration hanya terbaca vdpau dan vaapi yang mana driver hardware tidak ada, coba cek V4L2 apakah terinstall di servermu dengan :

<pre>
<code id="code-block">
ls /dev/video*
</code>
</pre>
# ANSWER : ```ls /dev/video* /dev/video0```
Perangkat /dev/video0 menandakan bahwa ada 2 hal yang terdeteksi oleh sistem, biasanya berupa :
 1. Mempunyai dukungan Hardware Video Encoder/Decoder (misalnya, pada chipset Amlogic S905X mempunyai GPU Mali-450 MP3 yang mendukung HARDWARE encoding Decoding)
 2. Rupanya servermu mendukung V4L2 untuk encoding decoding di GPU nya Amlogic s905x
Ini bisa jadi indikasi kegagalan/kesalahan proses compile di ffmpeg yang mana tidak sesuai spek/driver device mu.
# C. SOLUTION
Coba jalankan dulu ```ffmpeg -decoders | grep v4l2``` di CLI,
jika keluar

<pre>
<code id="code-block">
ffmpeg -decoders | grep v4l2
ffmpeg version 3.4.11-0ubuntu0.1 Copyright (c) 2000-2022 the FFmpeg developers
  built with gcc 7 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04)
  configuration: --prefix=/usr --extra-version=0ubuntu0.1 --toolchain=hardened --libdir=/usr/lib/aarch64-linux-gnu --incdir=/usr/include/aarch64-linux-gnu --enable-gpl --disable-stripping --enable-avresample --enable-avisynth --enable-gnutls --enable-ladspa --enable-libass --enable-libbluray --enable-libbs2b --enable-libcaca --enable-libcdio --enable-libflite --enable-libfontconfig --enable-libfreetype --enable-libfribidi --enable-libgme --enable-libgsm --enable-libmp3lame --enable-libmysofa --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libpulse --enable-librubberband --enable-librsvg --enable-libshine --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libssh --enable-libtheora --enable-libtwolame --enable-libvorbis --enable-libvpx --enable-libwavpack --enable-libwebp --enable-libx265 --enable-libxml2 --enable-libxvid --enable-libzmq --enable-libzvbi --enable-omx --enable-openal --enable-opengl --enable-sdl2 --enable-libdc1394 --enable-libdrm --enable-libiec61883 --enable-chromaprint --enable-frei0r --enable-libopencv --enable-libx264 --enable-shared
  libavutil      55. 78.100 / 55. 78.100
  libavcodec     57.107.100 / 57.107.100
  libavformat    57. 83.100 / 57. 83.100
  libavdevice    57. 10.100 / 57. 10.100
  libavfilter     6.107.100 /  6.107.100
  libavresample   3.  7.  0 /  3.  7.  0
  libswscale      4.  8.100 /  4.  8.100
  libswresample   2.  9.100 /  2.  9.100
  libpostproc    54.  7.100 / 54.  7.100
 V..... h263_v4l2m2m         V4L2 mem2mem H.263 decoder wrapper (codec h263)
 V..... h264_v4l2m2m         V4L2 mem2mem H.264 decoder wrapper (codec h264)
 V..... hevc_v4l2m2m         V4L2 mem2mem HEVC decoder wrapper (codec hevc)
 V..... mpeg1_v4l2m2m        V4L2 mem2mem MPEG1 decoder wrapper (codec mpeg1video)
 V..... mpeg2_v4l2m2m        V4L2 mem2mem MPEG2 decoder wrapper (codec mpeg2video)
 V..... mpeg4_v4l2m2m        V4L2 mem2mem MPEG4 decoder wrapper (codec mpeg4)
 V..... vc1_v4l2m2m          V4L2 mem2mem VC1 decoder wrapper (codec vc1)
 V..... vp8_v4l2m2m          V4L2 mem2mem VP8 decoder wrapper (codec vp8)
 V..... vp9_v4l2m2m          V4L2 mem2mem VP9 decoder wrapper (codec vp9)
</code>
</pre>
maka perangkat mu support hardware akseleration V4L2 yang support :
✅ H.263
✅ H.264
✅ HEVC (H.265)
✅ MPEG1
✅ MPEG2
✅ MPEG4
✅ VC1
✅ VP8
✅ VP9
Coba jalankan dulu Jellyfin dengan Config Hardware Akseleration : V4L2 (Video for Linux Versi 2)
![Screenshot 2025-03-13 111645](https://github.com/user-attachments/assets/b84a107b-3a3a-425c-9ee3-0fd9f6bca919)
Jika masih error coba liat log dengan ```journalctl | grep jellyfin```
# ANSWER
<pre>
<code id="code-block">
root@armbian-nis:~# journalctl | grep jellyfin
Mar 11 02:00:00 armbian-nis jellyfin[1350]: [02:00:00] [INF] DailyTrigger fired for task: Ekstrak Gambar Bagian
Mar 11 02:00:00 armbian-nis jellyfin[1350]: [02:00:00] [INF] Queuing task ChapterImagesTask
Mar 11 02:00:00 armbian-nis jellyfin[1350]: [02:00:00] [INF] Executing Ekstrak Gambar Bagian
Mar 11 02:00:00 armbian-nis jellyfin[1350]: [02:00:00] [INF] Ekstrak Gambar Bagian Completed after 0 minute(s) and 0 seconds
Mar 11 02:00:00 armbian-nis jellyfin[1350]: [02:00:00] [INF] ExecuteQueuedTasks
Mar 11 02:00:01 armbian-nis jellyfin[1350]: [02:00:01] [INF] Daily trigger for Ekstrak Gambar Bagian set to fire at 2025-03-12 02:00:00.000 +07:00, which is 23:59:58.9888410 from now.
Mar 11 06:26:54 armbian-nis jellyfin[1350]: [06:26:54] [INF] Authentication request for nongton has succeeded.
Mar 11 06:26:54 armbian-nis jellyfin[1350]: [06:26:54] [INF] Current/Max sessions for user nongton: 1/0
Mar 11 06:26:54 armbian-nis jellyfin[1350]: [06:26:54] [INF] Reissuing access token: 542b1e52f4ab4ef6ad428561e66837bc
Mar 11 06:26:54 armbian-nis jellyfin[1350]: [06:26:54] [INF] WS 192.168.2.100 request
Mar 11 06:29:56 armbian-nis jellyfin[1350]: [06:29:56] [INF] User policy for nongton. EnablePlaybackRemuxing: True EnableVideoPlaybackTranscoding: True EnableAudioPlaybackTranscoding: True
Mar 11 06:29:56 armbian-nis jellyfin[1350]: [06:29:56] [INF] RemoteClientBitrateLimit: 1500000, RemoteIp: 192.168.2.100, IsInLocalNetwork: True
Mar 11 06:29:56 armbian-nis jellyfin[1350]: [06:29:56] [INF] StreamBuilder.BuildVideoItem( Profile=Jellyfin Android, Path=/mnt/home/Movie/Terra formars 2016 BluRay-Ress.mp4, AudioStreamIndex=1, SubtitleStreamIndex=-1 ) => ( PlayMethod=DirectPlay, TranscodeReason=0 ) media:/videos/bf1f3dcf-ed13-4118-a6a4-205071bc0ad4/stream.mp4?MediaSourceId=bf1f3dcfed134118a6a4205071bc0ad4&Static=true&VideoCodec=h264&AudioCodec=aac&AudioStreamIndex=1&api_key=<token>&SubtitleMethod=Encode&Tag=f68f66f5b8b9f486a86fe6c231766739
Mar 11 06:30:31 armbian-nis jellyfin[1350]: [06:30:31] [INF] User policy for nongton. EnablePlaybackRemuxing: True EnableVideoPlaybackTranscoding: True EnableAudioPlaybackTranscoding: True
Mar 11 06:30:31 armbian-nis jellyfin[1350]: [06:30:31] [INF] RemoteClientBitrateLimit: 1500000, RemoteIp: 192.168.2.100, IsInLocalNetwork: True
Mar 11 06:30:31 armbian-nis jellyfin[1350]: [06:30:31] [INF] StreamBuilder.BuildVideoItem( Profile=Jellyfin Android, Path=/mnt/home/Movie/Terra formars 2016 BluRay-Ress.mp4, AudioStreamIndex=1, SubtitleStreamIndex=-1 ) => ( PlayMethod=Transcode, TranscodeReason=ContainerBitrateExceedsLimit ) media:/videos/bf1f3dcf-ed13-4118-a6a4-205071bc0ad4/master.m3u8?MediaSourceId=bf1f3dcfed134118a6a4205071bc0ad4&VideoCodec=h264,h264&AudioCodec=aac&AudioStreamIndex=1&VideoBitrate=589263&AudioBitrate=130737&AudioSampleRate=44100&MaxFramerate=23.976025&api_key=<token>&SubtitleMethod=Encode&RequireAvc=false&Tag=f68f66f5b8b9f486a86fe6c231766739&SegmentContainer=ts&BreakOnNonKeyFrames=False&h264-level=40&h264-videobitdepth=8&h264-profile=high&h264-audiochannels=2&aac-profile=lc&TranscodeReasons=ContainerBitrateExceedsLimit
Mar 11 06:30:33 armbian-nis jellyfin[1350]: [06:30:33] [INF] /usr/lib/jellyfin-ffmpeg/ffmpeg -analyzeduration 200M -ss 00:00:30.000 -f mov,mp4,m4a,3gp,3g2,mj2 -autorotate 0 -i file:"/mnt/home/Movie/Terra formars 2016 BluRay-Ress.mp4" -map_metadata -1 -map_chapters -1 -threads 0 -map 0:0 -map 0:1 -map -0:s -codec:v:0 libx264 -preset medium -crf 23 -maxrate 589263 -bufsize 1178526 -profile:v:0 high -level 40 -x264opts:0 subme=0:me_range=4:rc_lookahead=10:me=dia:no_chroma_me:8x8dct=0:partitions=none -force_key_frames:0 "expr:gte(t,30+n_forced*3)" -sc_threshold:v:0 0 -vf "setparams=color_primaries=bt709:color_trc=bt709:colorspace=bt709,scale=trunc(min(max(iw\,ih*a)\,640)/2)*2:trunc(ow/a/2)*2,format=yuv420p" -codec:a:0 copy -copyts -avoid_negative_ts disabled -max_muxing_queue_size 2048 -f hls -max_delay 5000000 -hls_time 3 -hls_segment_type mpegts -start_number 10 -hls_segment_filename "/var/lib/jellyfin/transcodes/f3b3c330ae464a285f4dd84f0ce79398%d.ts" -hls_playlist_type vod -hls_list_size 0 -y "/var/lib/jellyfin/transcodes/f3b3c330ae464a285f4dd84f0ce79398.m3u8"
Mar 11 06:30:33 armbian-nis jellyfin[1350]: [06:30:33] [ERR] Error processing request. URL GET /videos/bf1f3dcf-ed13-4118-a6a4-205071bc0ad4/hls1/main/10.ts.
Mar 11 06:30:33 armbian-nis jellyfin[1350]: System.UnauthorizedAccessException: Access to the path '/var/log/jellyfin/FFmpeg.Transcode-2025-03-11_06-30-33_bf1f3dcfed134118a6a4205071bc0ad4_df4bfe37.log' is denied.
Mar 11 06:30:33 armbian-nis jellyfin[1350]:  ---> System.IO.IOException: Permission denied
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    --- End of inner exception stack trace ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Interop.ThrowExceptionForIoErrno(ErrorInfo errorInfo, String path, Boolean isDirectory, Func`2 errorRewriter)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Interop.CheckIo(Error error, String path, Boolean isDirectory, Func`2 errorRewriter)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.Win32.SafeHandles.SafeFileHandle.Open(String path, OpenFlags flags, Int32 mode)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.Win32.SafeHandles.SafeFileHandle.Open(String fullPath, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at System.IO.Strategies.OSFileStreamStrategy..ctor(String path, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at System.IO.Strategies.FileStreamHelpers.ChooseStrategy(FileStream fileStream, String path, FileMode mode, FileAccess access, FileShare share, Int32 bufferSize, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Helpers.TranscodingJobHelper.StartFfMpeg(StreamState state, String outputPath, String commandLineArguments, HttpRequest request, TranscodingJobType transcodingJobType, CancellationTokenSource cancellationTokenSource, String workingDirectory)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Controllers.DynamicHlsController.GetDynamicSegment(StreamingRequestDto streamingRequest, Int32 segmentId)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Controllers.DynamicHlsController.GetHlsVideoSegment(Guid itemId, String playlistId, Int32 segmentId, String container, Int64 runtimeTicks, Int64 actualSegmentLengthTicks, Nullable`1 static, String params, String tag, String deviceProfileId, String playSessionId, String segmentContainer, Nullable`1 segmentLength, Nullable`1 minSegments, String mediaSourceId, String deviceId, String audioCodec, Nullable`1 enableAutoStreamCopy, Nullable`1 allowVideoStreamCopy, Nullable`1 allowAudioStreamCopy, Nullable`1 breakOnNonKeyFrames, Nullable`1 audioSampleRate, Nullable`1 maxAudioBitDepth, Nullable`1 audioBitRate, Nullable`1 audioChannels, Nullable`1 maxAudioChannels, String profile, String level, Nullable`1 framerate, Nullable`1 maxFramerate, Nullable`1 copyTimestamps, Nullable`1 startTimeTicks, Nullable`1 width, Nullable`1 height, Nullable`1 maxWidth, Nullable`1 maxHeight, Nullable`1 videoBitRate, Nullable`1 subtitleStreamIndex, Nullable`1 subtitleMethod, Nullable`1 maxRefFrames, Nullable`1 maxVideoBitDepth, Nullable`1 requireAvc, Nullable`1 deInterlace, Nullable`1 requireNonAnamorphic, Nullable`1 transcodingMaxAudioChannels, Nullable`1 cpuCoreLimit, String liveStreamId, Nullable`1 enableMpegtsM2TsMode, String videoCodec, String subtitleCodec, String transcodeReasons, Nullable`1 audioStreamIndex, Nullable`1 videoStreamIndex, Nullable`1 context, Dictionary`2 streamOptions)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at lambda_method1561(Closure , Object )
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfActionResultExecutor.Execute(IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Awaited|12_0(ControllerActionInvoker invoker, ValueTask`1 actionResultValueTask)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
Mar 11 06:30:33 armbian-nis jellyfin[1350]: --- End of stack trace from previous location ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeNextResourceFilter>g__Awaited|25_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Rethrow(ResourceExecutedContextSealed context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.InvokeFilterPipelineAsync()
Mar 11 06:30:33 armbian-nis jellyfin[1350]: --- End of stack trace from previous location ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Routing.EndpointMiddleware.<Invoke>g__AwaitRequestTask|6_0(Endpoint endpoint, Task requestTask, ILogger logger)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.ServerStartupMessageMiddleware.Invoke(HttpContext httpContext, IServerApplicationHost serverApplicationHost, ILocalizationManager localizationManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.WebSocketHandlerMiddleware.Invoke(HttpContext httpContext, IWebSocketManager webSocketManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.IpBasedAccessValidationMiddleware.Invoke(HttpContext httpContext, INetworkManager networkManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.LanFilteringMiddleware.Invoke(HttpContext httpContext, INetworkManager networkManager, IServerConfigurationManager serverConfigurationManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Authorization.Policy.AuthorizationMiddlewareResultHandler.HandleAsync(RequestDelegate next, HttpContext context, AuthorizationPolicy policy, PolicyAuthorizationResult authorizeResult)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.QueryStringDecodingMiddleware.Invoke(HttpContext httpContext)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Swashbuckle.AspNetCore.ReDoc.ReDocMiddleware.Invoke(HttpContext httpContext)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Swashbuckle.AspNetCore.SwaggerUI.SwaggerUIMiddleware.Invoke(HttpContext httpContext)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Swashbuckle.AspNetCore.Swagger.SwaggerMiddleware.Invoke(HttpContext httpContext, ISwaggerProvider swaggerProvider)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Authentication.AuthenticationMiddleware.Invoke(HttpContext context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.RobotsRedirectionMiddleware.Invoke(HttpContext httpContext)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.LegacyEmbyRouteRewriteMiddleware.Invoke(HttpContext httpContext)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.ResponseCompression.ResponseCompressionMiddleware.InvokeCore(HttpContext context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.ResponseTimeMiddleware.Invoke(HttpContext context, IServerConfigurationManager serverConfigurationManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.ExceptionMiddleware.Invoke(HttpContext context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]: [06:30:33] [INF] Stopping ffmpeg process with q command for /var/lib/jellyfin/transcodes/f3b3c330ae464a285f4dd84f0ce79398.m3u8
Mar 11 06:30:33 armbian-nis jellyfin[1350]: [06:30:33] [INF] /usr/lib/jellyfin-ffmpeg/ffmpeg -analyzeduration 200M -ss 00:00:30.000 -f mov,mp4,m4a,3gp,3g2,mj2 -autorotate 0 -i file:"/mnt/home/Movie/Terra formars 2016 BluRay-Ress.mp4" -map_metadata -1 -map_chapters -1 -threads 0 -map 0:0 -map 0:1 -map -0:s -codec:v:0 libx264 -preset medium -crf 23 -maxrate 589263 -bufsize 1178526 -profile:v:0 high -level 40 -x264opts:0 subme=0:me_range=4:rc_lookahead=10:me=dia:no_chroma_me:8x8dct=0:partitions=none -force_key_frames:0 "expr:gte(t,30+n_forced*3)" -sc_threshold:v:0 0 -vf "setparams=color_primaries=bt709:color_trc=bt709:colorspace=bt709,scale=trunc(min(max(iw\,ih*a)\,640)/2)*2:trunc(ow/a/2)*2,format=yuv420p" -codec:a:0 copy -copyts -avoid_negative_ts disabled -max_muxing_queue_size 2048 -f hls -max_delay 5000000 -hls_time 3 -hls_segment_type mpegts -start_number 10 -hls_segment_filename "/var/lib/jellyfin/transcodes/f3b3c330ae464a285f4dd84f0ce79398%d.ts" -hls_playlist_type vod -hls_list_size 0 -y "/var/lib/jellyfin/transcodes/f3b3c330ae464a285f4dd84f0ce79398.m3u8"
Mar 11 06:30:33 armbian-nis jellyfin[1350]: [06:30:33] [ERR] Error processing request. URL GET /videos/bf1f3dcf-ed13-4118-a6a4-205071bc0ad4/hls1/main/10.ts.
Mar 11 06:30:33 armbian-nis jellyfin[1350]: System.UnauthorizedAccessException: Access to the path '/var/log/jellyfin/FFmpeg.Transcode-2025-03-11_06-30-33_bf1f3dcfed134118a6a4205071bc0ad4_fa68309d.log' is denied.
Mar 11 06:30:33 armbian-nis jellyfin[1350]:  ---> System.IO.IOException: Permission denied
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    --- End of inner exception stack trace ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Interop.ThrowExceptionForIoErrno(ErrorInfo errorInfo, String path, Boolean isDirectory, Func`2 errorRewriter)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Interop.CheckIo(Error error, String path, Boolean isDirectory, Func`2 errorRewriter)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.Win32.SafeHandles.SafeFileHandle.Open(String path, OpenFlags flags, Int32 mode)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.Win32.SafeHandles.SafeFileHandle.Open(String fullPath, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at System.IO.Strategies.OSFileStreamStrategy..ctor(String path, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at System.IO.Strategies.FileStreamHelpers.ChooseStrategy(FileStream fileStream, String path, FileMode mode, FileAccess access, FileShare share, Int32 bufferSize, FileOptions options, Int64 preallocationSize)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Helpers.TranscodingJobHelper.StartFfMpeg(StreamState state, String outputPath, String commandLineArguments, HttpRequest request, TranscodingJobType transcodingJobType, CancellationTokenSource cancellationTokenSource, String workingDirectory)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Controllers.DynamicHlsController.GetDynamicSegment(StreamingRequestDto streamingRequest, Int32 segmentId)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Api.Controllers.DynamicHlsController.GetHlsVideoSegment(Guid itemId, String playlistId, Int32 segmentId, String container, Int64 runtimeTicks, Int64 actualSegmentLengthTicks, Nullable`1 static, String params, String tag, String deviceProfileId, String playSessionId, String segmentContainer, Nullable`1 segmentLength, Nullable`1 minSegments, String mediaSourceId, String deviceId, String audioCodec, Nullable`1 enableAutoStreamCopy, Nullable`1 allowVideoStreamCopy, Nullable`1 allowAudioStreamCopy, Nullable`1 breakOnNonKeyFrames, Nullable`1 audioSampleRate, Nullable`1 maxAudioBitDepth, Nullable`1 audioBitRate, Nullable`1 audioChannels, Nullable`1 maxAudioChannels, String profile, String level, Nullable`1 framerate, Nullable`1 maxFramerate, Nullable`1 copyTimestamps, Nullable`1 startTimeTicks, Nullable`1 width, Nullable`1 height, Nullable`1 maxWidth, Nullable`1 maxHeight, Nullable`1 videoBitRate, Nullable`1 subtitleStreamIndex, Nullable`1 subtitleMethod, Nullable`1 maxRefFrames, Nullable`1 maxVideoBitDepth, Nullable`1 requireAvc, Nullable`1 deInterlace, Nullable`1 requireNonAnamorphic, Nullable`1 transcodingMaxAudioChannels, Nullable`1 cpuCoreLimit, String liveStreamId, Nullable`1 enableMpegtsM2TsMode, String videoCodec, String subtitleCodec, String transcodeReasons, Nullable`1 audioStreamIndex, Nullable`1 videoStreamIndex, Nullable`1 context, Dictionary`2 streamOptions)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at lambda_method1561(Closure , Object )
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfActionResultExecutor.Execute(IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Awaited|12_0(ControllerActionInvoker invoker, ValueTask`1 actionResultValueTask)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
Mar 11 06:30:33 armbian-nis jellyfin[1350]: --- End of stack trace from previous location ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeNextResourceFilter>g__Awaited|25_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Rethrow(ResourceExecutedContextSealed context)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.InvokeFilterPipelineAsync()
Mar 11 06:30:33 armbian-nis jellyfin[1350]: --- End of stack trace from previous location ---
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Routing.EndpointMiddleware.<Invoke>g__AwaitRequestTask|6_0(Endpoint endpoint, Task requestTask, ILogger logger)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.ServerStartupMessageMiddleware.Invoke(HttpContext httpContext, IServerApplicationHost serverApplicationHost, ILocalizationManager localizationManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.WebSocketHandlerMiddleware.Invoke(HttpContext httpContext, IWebSocketManager webSocketManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.IpBasedAccessValidationMiddleware.Invoke(HttpContext httpContext, INetworkManager networkManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Jellyfin.Server.Middleware.LanFilteringMiddleware.Invoke(HttpContext httpContext, INetworkManager networkManager, IServerConfigurationManager serverConfigurationManager)
Mar 11 06:30:33 armbian-nis jellyfin[1350]:    at Microsoft.AspNetCore.Authorization.Policy.AuthorizationMiddlewareResultHandler.HandleAsync(RequestDelegate next, HttpContext context, AuthorizationPolicy policy, PolicyAuthorizationResult authorizeResult)
</code>
</pre>
masih menunjukan ```source error``` ini kendalanya saat dimatikan atau sesudah restart jellyfin/server.
Ya kita bahas untuk sementara jika urgent coba dulu pakai METODE directplay di jellyfin, tetapi Jellyfin akan gagal membaca URL HLS, jika kamu pingin menggunakan fitur Live TV / IPTV, karena HLS (m3u) harus di transcode ke format (.ts) lalu di encode/decode ke format h264,mpeg,HEVC.
# SOLUSI 
Dilihat dari log
1. Jellyfin menjalankan tugas harian
2. Autentikasi berhasil untuk user "nongton"
3. Masalah utama: Transcoding gagal karena izin akses
Ada error System.UnauthorizedAccessException yang menunjukkan bahwa Jellyfin tidak bisa mengakses path /var/log/jellyfin/FFmpeg.Transcode-2025-03-11_06-30-33_bf1f3dcfed134118a6a4205071bc0ad4_df4bfe37.log
Penyebabnya kemungkinan besar adalah izin (permissions) pada direktori log atau transcode
Jalankan perintah berikut untuk melihat siapa pemilik folder log dan transcode :

<pre>
<code id="code-block">
ls -lah /var/log/jellyfin
ls -lah /var/lib/jellyfin/transcodes
</code>
</pre>
Jika folder tersebut bukan milik user Jellyfin, ubah kepemilikannya:

<pre>
<code id="code-block">
sudo chown -R jellyfin:jellyfin /var/log/jellyfin
sudo chown -R jellyfin:jellyfin /var/lib/jellyfin/transcodes
</code>
</pre>
Dan pastikan ada izin baca/tulis:

<pre>
<code id="code-block">
sudo chmod -R 755 /var/log/jellyfin
sudo chmod -R 755 /var/lib/jellyfin/transcodes
</code>
</pre>
Setelah mengubah izin, restart Jellyfin ```sudo systemctl restart jellyfin```
Coba akses ulang video yang sebelumnya bermasalah dan lihat apakah masih ada error...

# ANSWER
![Screenshot 2025-03-13 113729](https://github.com/user-attachments/assets/ad1a96d2-8f7e-4154-ab54-6dab00bec8ca)
memutar film bisa sekarang pakai transcode yang mana bitrate aslinya film 1.3 Mbps 1080p SDR, bisa diseusaikan dengan perangkat dan jaringan melalui transcode jadi sebesar 420 kbps ke 426p.
![Screenshot 2025-03-13 114052](https://github.com/user-attachments/assets/aa623673-6a94-4097-98ba-a87e1c8c0dec)
memutar IPTV sekarang lancar, hanya saja ini pakai Software Akselerasi bukan Hardware Akselerasi.

# CLOSED SOLUTION
Itu berarti FFmpeg di sistemmu memang memiliki dukungan V4L2 mem2mem (m2m) decoder, tetapi tidak terdeteksi dalam daftar metode hardware acceleration di ```ffmpeg -hwaccels```
penyebab:
1. FFmpeg di sistemmu versi 3.4.11 (rilis lama, Ubuntu 18.04)
2. Driver V4L2 Belum Aktif Penuh : Meskipun /dev/video0 ada, bisa jadi V4L2 mem2mem tidak berjalan dengan baik
   coba : jalankan ```ls /dev/video* v4l2-ctl --list-formats-ext -d /dev/video0```
   jika ```Command 'v4l2-ctl' not found, but can be installed with: apt install v4l-utils root@armbian-nis:~# ```
   berarti itu penyebabanya , install dulu depedenci v41-utils untuk utilitas V4L2.
3. Dukungan V4L2 Mem2Mem Perlu Modul Kernel Tambahan
4. Compile ffmpeg tidak benar .
Karena sistemmu masih memakai FFmpeg 3.4.11, coba update ke versi lebih baru.
Namun, karena sistemmu berbasis Ubuntu 18.04, paket terbaru mungkin tidak tersedia langsung.
Solusi: Compile FFmpeg terbaru secara benar atau pakai PPA pihak ketiga :

<pre>
<code id="code-block">
sudo add-apt-repository ppa:savoury1/ffmpeg4
sudo apt update
sudo apt install ffmpeg
</code>
</pre>
solusi terakhir pakai metode software akselerasi  yang mana menggunakan libmali bukan V4L2, usahakan libmali ada di servermu:
jalankan ```dpkg -l | grep mali``` Jika GPU Mali berfungsi, akan muncul output seperti:
<pre>
<code id="code-block">
ii  gpg                                   2.2.4-1ubuntu1.6                       arm64        GNU Privacy Guard -- minimalist public key operations
ii  libmnl0:arm64                         1.0.4-2                                arm64        minimalistic Netlink communication library
</code>
</pre>
maka bisa digunakan untuk software akselerasi, hanya saja akan membebankan CPU bukan si GPU Mali-450 MP3, sehingga transcode terasa berat, suhu naik, sering cek saja di ```htop``` cli atau tambahkan cooler fan cpu & cooler fan eksternal di server mu.
Semoga bermanfaat artikel ini, silakan dishare.
