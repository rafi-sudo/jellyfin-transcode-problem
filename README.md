# jellyfin-transcode-problem
# JELLYFIN TRANSCODE PROBLEM

# A. DESCRIBE
Jellyfin membutuhkan hardware atau software decoding untuk melakukan transcode. Server saya pakai HG680P dengan Armbian Ubuntu 18 dan prosesor Amlogic S905X, ada beberapa kemungkinan kenapa transcode tidak jalan:
1. Amlogic S905X punya hardware decoding (VPU), tapi driver untuk Armbian mungkin tidak tersedia atau tidak diaktifkan.
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
 1. Mempunyai dukungan Hardware Video Encoder/Decoder (misalnya, pada chipset Amlogic S905X mempunyai VPU yang mendukung HARDWARE encoding Decoding)
 2. Rupanya servermu mendukung V4L2 untuk encoding decoding di VPU nya Amlogic s905x
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

