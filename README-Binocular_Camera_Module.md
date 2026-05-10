# Binocular Camera Module, Dual IMX219, 8 Megapixels, Applicable for Jetson Nano and Raspberry Pi, Stereo Vision, Depth Vision


* 링크 :
   * https://www.waveshare.com/product/ai/cameras/imx219-83-stereo-camera.htm?___SID=U
   * https://www.waveshare.com/wiki/IMX219-83_Stereo_Camera
 
## IMX219-83 Stereo Camera

Overview
Specification
8 Megapixels
Sensor: Sony IMX219
Resolution: 3280 × 2464 (per camera)
Lens specifications:
CMOS size: 1/4inch
Focal Length: 2.6mm
Angle of View: 83/73/50 degree (diagonal/horizontal/vertical)
Distortion: <1%
Baseline Length: 60mm
ICM20948:
Accelerometer:
Resolution: 16-bit
Measuring Range (configurable): ±2, ±4, ±8, ±16g
Operating Current: 68.9uA
Gyroscope:
Resolution: 16-bit
Measuring Range (configurable): ±250, ±500, ±1000, ±2000°/sec
Operating Current: 1.23mA
Magnetometer:
Resolution: 16-bit
Measuring Range: ±4900μT
Operating Current: 90uA
Dimension: 24mm × 85mm
Working With Jetson Nano
Hardware Connection
Please use two camera cables, facing their metal sides to Jetson Nano's heatsink and insert them to the CSI interfaces.
Boot the Jetson Nano.
Software setting
Power on Jetson Nano and open the Terminal (Ctrl+ALT+T).
Check the video devices with the following command:
ls /dev/video*
Check if both video0 and video1 are detected.
Test video0:
DISPLAY=:0.0 nvgstcapture-1.0 --sensor-id=0
Test video1:
 DISPLAY=:0.0 nvgstcapture-1.0 --sensor-id=1
If the camera's imaging is too red, you can operate as below:
1. Download the "camera-override.isp" file and unzip it to the specified file folder.

wget https://files.waveshare.com/upload/e/eb/Camera_overrides.tar.gz
tar zxvf Camera_overrides.tar.gz 
sudo cp camera_overrides.isp /var/nvidia/nvcam/settings/
2. Install the file:

sudo chmod 664 /var/nvidia/nvcam/settings/camera_overrides.isp
sudo chown root:root /var/nvidia/nvcam/settings/camera_overrides.isp
【Note】
- 12 of NV12 is a number, not a letter.
- The test screen is output to the HDMI or DP screen, so the screen must be connected to the Jetson Nano when testing.
How to Use IMU Sensor:
The camera module has an onboard ICM20948 9-axis sensor. Users can connect it to the I2C pin of the Jetson Nano with the provided 4-pin cable.

To test the sensor, respectively connect the SDA and SCL pins of the camera to pins 3 and 5 on the Jetson Nano.
To read the sensor data normally, you only need to connect the SDA and SCL pins.
IMX219 Jetson Nano.png

Open the terminal and download the sample demo, then test:
wget https://files.waveshare.com/upload/e/eb/D219-9dof.tar.gz
tar zxvf D219-9dof.tar.gz
cd D219-9dof/07-icm20948-demo
make
./ICM20948-Demo
Please try to rotate the camera, and you can see the output value changes.

Test with Compute Module
The IMX219 series camera can be used in the same way as other Raspberry Pi cameras, if you use Raspberry Pi 5, you need to use the supplied 22PIN cable to connect to the IMX219 series camera.

libcamera/libcamera Usage
Introduction
After the Bullseye version, the underlying Raspberry Pi driver for the Raspberry Pi image has been switched from Raspicam to libcamera. Libcamera is an open-source software stack (referred to as a driver later for ease of understanding) that is convenient for third-party porting and developing their own camera drivers. As of December 11, 2023, the official picamera2 library has been provided for libcamera, making it easier for users to call Python demos.

Call Camera
The libcamera software stack provides six commands for users to preview and test the camera interface.

libcamera-hello
This is a simple "hello world" program that previews the camera and displays the camera image on the screen.
Example

libcamera-hello
This command will preview the camera on the screen for about 5 seconds. The user can use the "-t <duration>" parameter to set the preview time, where the unit of <duration> is milliseconds. If it is set to 0, it will keep previewing all the time. For example:

libcamera-hello -t 0
Tuning File
The libcamera driver of the Raspberry Pi will call a tuning file for different camera modules. The tuning file provides various parameters. When calling the camera, libcamera will call the parameters in the tuning file, and process the image in combination with the algorithm. The final output is the preview screen. Since the libcamera driver can only automatically receive the signal of the chip, the final display effect of the camera will also be affected by the entire module. The use of the tuning file is to flexibly handle the cameras of different modules and adjust to improve the image quality.
If the output image of the camera is not ideal after using the default tuning file, the user can adjust the image by calling the custom tuning file. For example, if you are using the official NOIR version of the camera, the NOIR camera may require different white balance parameters compared with the regular Raspberry Pi Camera V2. In this case, you can switch by calling the tuning file.

libcamera-hello --tuning-file /usr/share/libcamera/ipa/raspberrypi/imx219_noir.json
Users can copy the default tuning files and modify them according to their needs.
Note: The use of tuning files applies to other libcamera commands, and will not be introduced in subsequent commands.

Preview Window
Most libcamera commands will display a preview window on the screen. Users can customize the title information of the preview window through the --info-text parameter, and can also call some camera parameters through %directives and display them on the window.
For example, if you use HQ Camera: You can display the focal length of the camera on the window through --info-txe "%focus".

libcamera-hello --info-text "focus %focus".
Note: For more information on parameter settings, please refer to the following chapters.

libcamera-jpeg
libcamera-jpeg is a simple static picture shooting program, different from the complex functions of libcamera-still, libcamera-jpeg code is more concise and has many of the same functions to complete picture shooting.
Take a JPEG image of full pixel

libcamera-jpeg -o test.jpg
This shooting command will display a preview serial port for about 5 seconds, and then shoot a full-pixel JPEG image and save it as test.jpg.
Users can set the preview time through the -t parameter and can set the resolution of the captured image through --width and --height. E.g.:

libcamera-jpeg -o test.jpg -t 2000 --width 640 --height 480
Exposure control
All libcamera commands allow the user to set the shutter time and gain themselves, such as:

libcamera-jpeg -o test.jpg -t 2000 --shutter 20000 --gain 1.5
This command will capture an image with 20ms exposure and camera gain set to 1.5x. The gain parameter set will first set the analog gain parameter inside the sensor chip. If the set gain exceeds the maximum built-in analog gain value of the driver, the maximum analog gain of the chip will be set first, and then the remaining gain multiples will be used as digital gains to take effect.
Note: The digital gain is realized by ISP (image signal processing), not directly adjusting the built-in register of the chip. Under normal circumstances, the default digital gain is close to 1.0, unless there are the following three situations.

Overall gain parameter requirements, that is, when the analog gain cannot meet the set gain parameter requirements, the digital gain will be needed for compensation.
One of the color gains is less than 1 (the color gain is achieved by digital gain), in this case, the final gain is stabilized at 1/min(red_gain, blue_gain), that is, a uniform digital gain is applied, and is the gain value for one of the color channels (not the green channel).
AEC/AGC was modified. If there is a change in AEC/AGC, the digital gain will also change to a certain extent to eliminate any fluctuations, this change will be quickly restored to the "normal" value.
The Raspberry Pi's AEC/AGX algorithm allows the program to specify exposure compensation, which is to adjust the brightness of the image by setting the aperture value, for example:

libcamera-jpeg --ev -0.5 -o darker.jpg
libcamera-jpeg --ev 0 -o normal.jpg
libcamera-jpeg --ev 0.5 -o brighter.jpg
libcamera-still
libcamera-still and libcamera-jpeg are very similar, the difference is that libcamera inherits more functions of raspistill. As before, users can take a picture with the following command.
Test Command

libcamera-still -o test.jpg
Encoder
libcamera-still supports image files in different formats, can support png and bmp encoding, and also supports saving binary dumps of RGB or YUV pixels as files without encoding or in any image format. If you save RGB or YUV data directly, the program must understand the pixel arrangement of the file when reading such files.

libcamera-still -e png -o test.png
libcamera-still -e bmp -o test.bmp
libcamera-still -e rgb -o test.data
libcamera-still -e yuv420 -o test.data
Note: The format of image saving is controlled by the -e parameter. If the -e parameter is not called, it will be saved in the format of the output file name by default.
Raw Image Capture
The raw image is the image output by the direct image sensor without any ISP or CPU processing. For color camera sensors, the output format of the raw image is generally Bayer. Note that the raw image is different from the bit-encoded RGB and YUV images we said earlier, and RGB and YUV are also ISP-processed images.
A command to take a raw image:

libcamera-still -r -o test.jpg
The raw image is generally saved in DNG (Adobe Digital Negative) format, which is compatible with most standard programs, such as dcraw or RawTherapee. The raw image will be saved as a file of the same name with the .dng suffix, for example, if you run the above command, it will be saved as a test.dng, and generate a jpeg file at the same time. The DNG file contains metadata related to image acquisition, such as white balance data, ISP color matrix, etc. The following is the metadata encoding information displayed by the exiftool:

File Name                       : test.dng
Directory                       : .
File Size                       : 24 MB
File Modification Date/Time     : 2021:08:17 16:36:18+01:00
File Access Date/Time           : 2021:08:17 16:36:18+01:00
File Inode Change Date/Time     : 2021:08:17 16:36:18+01:00
File Permissions                : rw-r--r--
File Type                       : DNG
File Type Extension             : dng
MIME Type                       : image/x-adobe-dng
Exif Byte Order                 : Little-endian (Intel, II)
Make                            : Raspberry Pi
Camera Model Name               : /base/soc/i2c0mux/i2c@1/imx477@1a
Orientation                     : Horizontal (normal)
Software                        : libcamera-still
Subfile Type                    : Full-resolution Image
Image Width                     : 4056
Image Height                    : 3040
Bits Per Sample                 : 16
Compression                     : Uncompressed
Photometric Interpretation      : Color Filter Array
Samples Per Pixel               : 1
Planar Configuration            : Chunky
CFA Repeat Pattern Dim          : 2 2
CFA Pattern 2                   : 2 1 1 0
Black Level Repeat Dim          : 2 2
Black Level                     : 256 256 256 256
White Level                     : 4095
DNG Version                     : 1.1.0.0
DNG Backward Version            : 1.0.0.0
Unique Camera Model             : /base/soc/i2c0mux/i2c@1/imx477@1a
Color Matrix 1                  : 0.8545269369 -0.2382823821 -0.09044229197 -0.1890484985 1.063961506 0.1062747385 -0.01334283455 0.1440163847 0.2593136724
As Shot Neutral                 : 0.4754476844 1 0.413686484
Calibration Illuminant 1        : D65
Strip Offsets                   : 0
Strip Byte Counts               : 0
Exposure Time                   : 1/20
ISO                             : 400
CFA Pattern                     : [Blue, Green][Green, Red]
Image Size                      : 4056x3040
Megapixels                      : 12.3
Shutter Speed                   : 1/20
Long Exposure
If we want to take a super long exposure picture, we need to disable AEC/AGC and white balance, otherwise, these algorithms will cause the picture to wait for a lot of frame data when it converges. Disabling these algorithms requires another explicit value to be set. Additionally, the user can skip the preview process with the --immediate setting.
Here is the command to take an image with an exposure of 100 seconds:

libcamera-still -o long_exposure.jpg --shutter 100000000 --gain 1 --awbgains 1,1 --immediate
Remarks: Reference table for the longest exposure time of several official cameras.

Module	Maximum exposure time (s)
V1 (OV5647)	6
V2 (IMX219)	11.76
V3 (IMX708)	112
HQ (IMX477)	670
libcamera-vid
libcamera-vid is a video recording demo that uses the Raspberry Pi hardware H.264 encoder by default. After the program runs, a preview window will be displayed on the screen, and simultaneously the bitstream encoding will be output to the specified file. For example, record a 10s video.

libcamera-vid -t 10000 -o test.h264
If you want to view the video, you can use vlc to play it.

vlc test.h264
Note: The recorded video stream is unpacked. Users can use --save-pts to set the output timestamp to facilitate the subsequent conversion of the bit stream to other video formats.

libcamera-vid -o test.h264 --save-pts timestamps.txt
If you want to output the mkv file, you can use the following command:

mkvmerge -o test.mkv --timecodes 0:timestamps.txt test.h264
Encoder
Raspberry Pi supports JPEG format and YUV420 without compression and format:

libcamera-vid -t 10000 --codec mjpeg -o test.mjpeg
libcamera-vid -t 10000 --codec yuv420 -o test.data
The --codec option sets the output format, not the output file extension.
Use the --segment parameter to split the output file into segments (unit is ms), which is suitable for JPEG files that need to split the JPEG video stream into separate short (about 1ms) JPEG files.

libcamera-vid -t 10000 --codec mjpeg --segment 1 -o test%05d.jpeg
UDP Video Streaming Transmission
UDP can be used for video streaming transmission, and the Raspberry Pi runs (server):

libcamera-vid -t 0 --inline -o udp://<ip-addr>:<port>
Among them, <ip-addr> needs to be replaced with the actual client IP address or multicast address. On the client (client), enter the following commands to obtain and display the video stream (you can use one of the two commands):

vlc udp://@:<port> :demux=h264
vlc udp://@:<port> :demux=h264
Note: The port needs to be the same as the one you set on the Raspberry Pi.
TCP Video Streaming Transmission
You can use TCP for video streaming Transmission, and the Raspberry Pi runs (server):

libcamera-vid -t 0 --inline --listen -o tcp://0.0.0.0:<port>
The client runs:

vlc tcp/h264://<ip-addr-of-server>:<port> #Select one of the two commands
ffplay tcp://<ip-addr-of-server>:<port> -vf "setpts=N/30" -fflags nobuffer -flags low_delay -framedrop
RTSP Video Streaming Transmission
On the Raspberry Pi, vlc is usually used to process the RTSP video stream:

libcamera-vid -t 0 --inline -o - | cvlc stream:///dev/stdin --sout '#rtp{sdp=rtsp://:8554/stream1}' :demux=h264
On the playback side, you can run any of the following commands:

vlc rtsp://<ip-addr-of-server>:8554/stream1
ffplay rtsp://<ip-addr-of-server>:8554/stream1 -vf "setpts=N/30" -fflags nobuffer -flags low_delay -framedrop
In all preview commands, if you want to turn off the preview window on the Raspberry Pi, you can use the parameter -n (--nopreview) to set it. Also, pay attention to the setting of the --inline parameter. Changing the setting will force the header information of the video stream to be included in each I (intra) frame. This setting allows the client to correctly parse the video stream even if the video header is lost.
High Frame Rate Mode
If you use the libcamera-vid command to record high frame rate video (generally higher than 60fps) while reducing frame loss, you need to pay attention to the following points:

The target level of H.264 needs to be set to 4.2, which can be set with the --level 4.2 parameter.
The color noise reduction function must be turned off, which can be set with the --denoise cdn_off parameter.
If the set frame rate is higher than 100fps, close the preview window to release more CPU resources and avoid frame loss, which can be set with the -n parameter.
It is recommended to add the setting force_turbo=1 in the /boot/config.txt file to ensure that the CPU clock will not be limited in the video stream.
Adjust the ISP output resolution, use -width 1280 --height 720 to set the resolution, or set it to a lower resolution, depending on the camera model.
If you are using Pi 4 or higher-performance model, you can add the setting gpu_freq=550 or higher in the /boot/config.txt file to overclock the motherboard GPU to achieve higher performance.
For example, record 1280x720 120fps video.

libcamera-vid --level 4.2 --framerate 120 --width 1280 --height 720 --save-pts timestamp.pts -o video.264 -t 10000 --denoise cdn_off -n
libcamera-raw
Libcamera-raw is similar to a video recording demo. The difference is, libcamera-raw records the Bayer format data output by the direct sensor, that is, the raw image data. Libcamera-raw doesn't show a preview window. For example, record a 2-second piece of raw data.

libcamera-raw -t 2000 -o test.raw
The demo will directly dump the raw frame without format information, the program will directly print the pixel format and image size on the terminal, and the user can view the pixel data according to the output data.
By default, the program will save the raw frame as a file, the file is usually large, and the user can divide the file by the --segement parameter.

libcamera-raw -t 2000 --segment 1 -o test%05d.raw
If the memory is large (such as using SSD), libcamera-raw can write the official HQ Camera data (about 18MB per frame) to the hard disk at a speed of about 10 frames per second. To achieve this speed, the demo writes the unformatted raw frames, there is no way to save them as DNG files like libcamera-still does. If you want to ensure that there are no dropped frames, you can use --framerate to reduce the frame rate.

libcamera-raw -t 5000 --width 4056 --height 3040 -o test.raw --framerate 8
Common Command Setting Options
Common command setting options apply to all libcamera commands:

--help, -h
Print program help information, you can print the available setting options for each program command, and then exit.

--version
Print the software version, print the software version of libcamera and libcamera-app, then exit.

--list-cameras
Display the recognized supported cameras. For example:

Available cameras
-----------------
0 : imx219 [3280x2464] (/base/soc/i2c0mux/i2c@1/imx219@10)
    Modes: 'SRGGB10_CSI2P': 640x480 [206.65 fps - (1000, 752)/1280x960 crop]
                             1640x1232 [41.85 fps - (0, 0)/3280x2464 crop]
                             1920x1080 [47.57 fps - (680, 692)/1920x1080 crop]
                             3280x2464 [21.19 fps - (0, 0)/3280x2464 crop]
           'SRGGB8' : 640x480 [206.65 fps - (1000, 752)/1280x960 crop]
                      1640x1232 [41.85 fps - (0, 0)/3280x2464 crop]
                      1920x1080 [47.57 fps - (680, 692)/1920x1080 crop]
                      3280x2464 [21.19 fps - (0, 0)/3280x2464 crop]
1 : imx477 [4056x3040] (/base/soc/i2c0mux/i2c@1/imx477@1a)
    Modes: 'SRGGB10_CSI2P': 1332x990 [120.05 fps - (696, 528)/2664x1980 crop]
           'SRGGB12_CSI2P': 2028x1080 [50.03 fps - (0, 440)/4056x2160 crop]
                             2028x1520 [40.01 fps - (0, 0)/4056x3040 crop]
                             4056x3040 [10.00 fps - (0, 0)/4056x3040 crop]
According to the printed information, the IMX219 camera has a suffix of 0, and the IM new 477 camera has a suffix of 1. When calling the camera, you can specify the corresponding suffix.

--camera
Specify the camera, and the corresponding suffix can refer to the print information of the command --list-camera.
For example: libcamera-hello -c config.txt
In the setting file, set parameters one line at a time, in the format of key=value:

timeout=99000
verbose=
--config,	-c
Under normal circumstances, we can directly set the camera parameters through commands. Here we use the --config parameter to specify the setting file and directly read the setting parameters in the file to set the camera preview effect.

--timeout, -t
The "-t" option sets the runtime of the libcamera demo. If the video recording command is run, the timeout option sets the recording duration. If the image capture command is run, the timeout sets the preview time before the image is captured and output.
If "timeout" is not set when running the libcamera demo, the default timeout value is 5000 (5 seconds). If the timeout is set to 0, the demo will continue to run.
Example: libcamera-hello -t 0

--preview, -p
"-p" sets the size and position of the preview window (the qualified settings are valid in both X and DRM version windows), and the format is --preview <x. y, w, h> where "x", "y" sets the preview window coordinates, "w" and "h" set the width and length of the preview window.
The settings of the preview serial port will not affect the resolution and aspect ratio of the camera image preview. The demo will scale the preview image to display in the preview window and adapt it according to the original image aspect ratio.
Example: libcamera-hello -p 100,100,500,500

--fullscreen, -f
The "-f" option sets the preview window full screen, the preview window and the border in full-screen mode. Like "-p", it does not affect the resolution and aspect ratio, and will automatically adapt.
Example: libcamera-still -f -o test.jpg

--qt-preview
Using the preview window based on the QT framework, this setting is not recommended under normal circumstances, because the preview demo will not use zero-copy buffer sharing and GPU acceleration, which will occupy more resources. The QT preview window supports X forwarding (the default preview program does not).
The Qt preview serial port does not support the "--fullscreen" setting option. If the user wants to use the Qt preview, it is recommended to keep a small preview window to avoid excessive resource usage and affecting the normal operation of the system.
Example: libcamera-hello --qt-preview

--nopreview, -n
Images are not previewed. This setting will turn off the image preview function.
Example: libcamera-hello -n

--info-text
Set the title and information display of the preview window (only available in the X graphics window) using the format --info-text <string>. When calling this option, multiple parameters can be set, and the parameters are usually called in the % command format. The demo will call the corresponding value in the graphics metadata according to the instructions.
If no window info is specified, the default --info-text is set to "#%frame (%fps fps) exp %exp ag %ag dg %dg" .
Example: libcamera-hello --info-test "Focus measure: %focus
Available parameters:

Instructions	Instructions
%frame	Frame sequence number
%fps	Instantaneous frame rate
%exp	Shutter speed when capturing the image, in ms
%ag	Image analog gain controlled by the sensor chip
%dg	Image value gain controlled by ISP
%rg	Gain of the red component of each pixel
%bg	Gain of the blue component of each pixel
%focus	Corner detection of the image, the larger the value, the clearer the image
%lp	Diopter of the current lens (1/distance in meters)
%afstate	Autofocus state (idle, scanning, focused, failed)
--width
--height
These two parameters set the width and height of the image respectively. For libcamera-still, libcamera-jpeg, and libcamera-vid commands, these two parameters can set the resolution of the output image/video.
If the libcamera-raw command is used, these two parameters will affect the size of the obtained metadata frame. The camera has a 2 x 2 block reading mode. If the set resolution is smaller than the split mode, the camera will obtain the metadata frame according to the 2 x 2 block size.
libcamera-hello cannot specify the resolution.
Example:
libcamera-vid -o test.h264 --width 1920 --height 1080 Record a 1080p video
libcamera-still -r -o test.jpg --width 2028 --height 1520 Take a 2028 x 1520 JPEG image

--viewfinder-width
--viewfinder-height
This setting option is also used to set the resolution of the image, the difference is only the image size of the preview. It does not affect the final output image or video resolution.The size of the preview image will not affect the size of the preview window and it will be adapted according to the window.
Example: libcamera-hello --viewfinder-width 640 --viewfinder-height 480

--rawfull
This setting forces the sensor chip to activate --width and --height settings to output still images and video in full-resolution reading mode. This setting libcamera-hello is invalid.
With this setting, the framerate will be affected. In full-resolution mode, frame reading will be slower.
Example: libcamera-raw -t 2000 --segment 1 --rawfull -o test%03d.raw
The example command will capture multiple Metadata frames in full-resolution mode. If you are using an HQ camera, the size of each frame is 18MB, and if --rawfull is not set, the HQ camera defaults to 2 x 2 mode, and the data size of each frame is only 4.5MB.

--mode
This parameter is more general than rawfull. It is used to set the camera mode. When using it, you need to specify the width, height, bit depth and packing mode, and separate them with colons. The set value does not have to be completely accurate, the system will automatically match the closest value, and the bit depth and packing mode can be set (the default is 12 and P means packing).

4056:3040:12:P - 4056x3040 resolution, 12 bits per pixel, packing. Packing means that the raw image data will be packed in the buffer. In this case, two pixels will only occupy 3 bytes, which can save memory.
1632:1224:10 - 1632x1224 resolution, 10 bits per pixel, packing by default. In 10-bit packing mode, 4-pixel data will occupy 5 bytes.
2592:1944:10:U - 2592x1944 resolution, 10 bits per pixel, no packing. In the case of unpacking, each pixel will occupy 2 bytes of memory, in this case, the highest 6 bits will be set to 0.
3262:2448 - 3264x2448 resolution, 12 bits and packing mode are used by default. However, if the camera model, such as Camera V2 (IMX219), does not support 12-bit mode, the system will automatically select 10-bit mode.
--viewfinder-mode       #Specify sensor mode, given as <width>:<height>:<bit-depth>:<packing>
The --mode parameter is used to set the camera mode when recording video and shooting still images. If you want to set it when previewing, you can use the --viewfinder-mode parameter.

--lores-width
--lores-height
These two options set low-resolution images. The low-resolution data stream compresses the image, causing the aspect ratio of the image to change. When using libcamera-vid to record video, if a low resolution is set, functions such as color image denoising will be disabled.
Example: libcamera-hello --lores-width 224 --lores-height 224
Note that low-resolution settings are often used in conjunction with image postprocessing, otherwise it has little effect.

--hflip #Flip the image horizontally
--vflip #Flip the image vertically
--rotation #Flip the image horizontally or vertically according to the given angle <angle>
These three options are used to flip the image. The parameters of --rotation currently only support 0 and 180, which are equivalent to --hflip and --vflip.
Example: libcamera-hello --vflip --hflip

--roi #Crop image <x, y, w, h>
The --roi parameter allows the user to crop the image area they want according to the coordinates from the complete image provided by the sensor, that is digital scaling, note that the coordinate values should be within the valid range. For example --roi 0, 0, 1, 1 is an invalid instruction.
Example: libcamera-hello --roi 0.25,0.25,0.5,0.5
The example command will crop 1/4 of the image from the center of the image.

--hdr #Run the camera in HDR mode (supported cameras only)
The --hdr parameter is used to set the wide dynamic mode of the camera. This setting will only take effect if the camera supports a wide dynamic range. You can use --list-camera to see if the camera supports hdr mode.

--sharpness #Set the sharpness of the image <number>
Adjust the sharpness of the image by the value of <number>. If set to 0, sharpening will be applied. If you set a value above 1.0, an extra sharpening amount will be used.
Example: libcamera-still -o test.jpg --sharpness 2.0

--contrast #Set image contrast <number>
Example: libcamera-still -o test.jpg --contrast 1.5

--brightness #Set image brightness <number>
The setting range is -1.0 ~ 1.0
Example: libcamera-still -o test.jpg --brightness 0.2

--saturation #Set image color saturation <number>
Example: libcamera-still -o test.jpg --saturation 0.8

--ev #Set EV compensation <number>
Set the EV compensation of the image in units of aperture, the setting range is -10 ~ 10, the default value is 0. The program runs using the AEC/AGC algorithm.
Example: libcamera-still -o test.jpg --ev 0.3

--shutter #Set the exposure time, the unit is ms <number>
Note: If the frame rate of the camera is too fast, it may not work according to the set shutter time. If this happens, you can try to use --framerate to reduce the frame rate.
Example: libcamera-hello --shutter 30000

--gain #Set gain value (combination of numerical gain and analog gain) <number>
--analoggain #--gain synonym
--analoggain is the same as --gain, the use of analoggain is only for compatibility with raspicam programs.

--metering #Set metering mode <string>
Set the metering mode of the AEC/AGC algorithm, the available parameters are:

centre - Center metering (default)
spot - spot metering
average - average or full frame metering
custom - custom metering mode, can be set via tuning file
Example: libcamera-still -o test.jpg --metering spot

--exposure #Set exposure profile <string>
The exposure mode can be set to normal or sport. The report profile for these two modes does not affect the overall exposure of the image, but in the case of sport mode, the program will shorten the exposure time and increase the gain to achieve the same exposure effect.

sport: Short exposure, larger gains
normal: Normal expoeure, normal gains
long: Long exposure, smaller gains
Example: libcamera-still -o test.jpg --exposure sport

--awb #Set white balance mode <string>
Available white balance modes:

Mode	Color Temperature
auto	2500K ~ 8000K
incadescent	2500K ~ 3000K
tungsten	3000K ~3500K
fluorescent	4000K ~ 4700K
indoor	3000K ~ 5000K
daylight	5500K ~ 6500K
cloudy	7000K ~ 8500K
custom	Custom range, set via tuning file
Example: libamera-still -o test.jpg --awb tungsten

--awbgains #Set a fixed color gain <number,number>
Set red and blue gain.
Example: libcamera-still -o test.jpg --awbgains 1.5, 2.0

--denoise #Set denoising mode <string>
Supported denoising modes:

auto - Default mode, use standard spatial denoising. If it is a video, fast color denoising will be used, and high-quality color denoising will be used when taking still images. Previewing images will not use any color denoising.
off - Turn off spatial denoising and color denoising.
cdn_off - Turn off color denoising.
cdn_fast - Use fast color denoising.
cdn_hq - Use high-quality color denoising, not suitable for video recording.
Example: libcamera-vid -o test.h264 --denoise cdn_off

--tuning-file #Specify camera tuning file <string>
For more instructions on tuning files, you can refer to official tutorial.
Example: libcamera-hello --tuning-file ~/my~camera-tuning.json

--autofocus-mode #Specify the autofocus mode <string>
Set the autofocus mode.

default - By default, the camera will use continuous autofocus mode, unless --lens-position or --autofocus-on-capture manual focus is set.
manual - Manual focus mode, the focus position can be set by --lens-position.
auto - Focusing will only be done once when the camera is turned on, and will not be adjusted in other cases. (If you use the libcamera-still command, only when --autofocus-on-capture is used, it will focus once before taking a photo).
continuous - The camera will automatically adjust the focus position according to the scene changes.
--autofocus-range #Specify the autofocus range <string>
Set the autofocus range.

normal -- The default mode, from nearest to infinity.
macro - The macro mode, only focus on nearby objects.
full - The full distance mode, adjusted to infinity for the closest object.
--autofocus-speed #Specify the autofocus speed <string>
Set the focus speed.

normal - The default item, normal speed.
fast - The fast focus mode.
--autofocus-window   --autofocus-window
To display the focus window, you need to set x, y, width and height, and the coordinate value setting is based on the ratio of the image. For example --autofocus-window 0.25,0.25,0.5,0.5 would set a window that is half the size of the image and centered.

--lens-position #Set the lens to a given position <string>
Set the focus position.

0.0 -- Set the focus position to infinity.
number -- Set the focus position to 1/number, number is any value you set, for example, if you set 2, it means that it will focus on the position of 0.5m.
default -- Focus on the default position relative to the hyperfocal distance of the lens.
--output, -o #Output file name <string>
Set the filename of the output image or video. In addition to setting the file name, you can also specify the output udp or tcp server address to output the image to the server. If you are interested, you can check the relevant setting instructions of the subsequent tcp and udp.
Example: libcamera-vid -t 100000 -o test.h264

--wrap #Wrap the output file counter <number>
Example: libcamera-vid -t 0 --codec mjpeg --segment 1 --wrap 100 -o image%d.jpg

--flush #Flush the output file immediately
The --flush parameter will immediately update each frame of the image to the hard disk at the same time as it is written, reducing latency.
Example: libcamera-vid -t 10000 --flush -o test.h264

Still Image Shooting Setting Parameters
--quality, -q            #Set JPEG image quality <0 ~ 100>
--exif, -x               #Add extra EXIF flags
--timelapse              #Time interval of time-lapse photography, the unit is ms
--framestart             #Start value of frame count
--datetime               #Name the output file with date format
--timestamp              #Name the output file with the system timestamp
-- restart               #Set the JPEG restart interval
--keypress, -k           #Set the Enter key photo mode
--signal, -s             #Set the signal to trigger the photo mode
--thumb                  #Set thumbnail parameters <w:h:q>
--ebcoding, -e           #Set the image encoding type: jpg/png/bmp/rgb/yuv420
--raw, -r                #Save raw image
--latest                 #Associate symbols to the latest saved file
--autofocus-on-capture   #Set to do a focus action before taking a photo
Video Recording Image Setting Parameters
--quality, -q   #Set JPEG commands <0 - 100>
--bitrate, -b   #Set H.264 bitrate
--intra, -g     #Set the internal frame period (only supports H.264)
--profile       #Set H.264 configuration
--level         #Set H.264 level
--codec         #Set encoding type h264 / mjpeg / yuv420
--keypress, -k  #Set Enter key to pause and record
--signal, -s    #Set signal pause and record
--initial       #Start the program in the recording or paused state
--split         #Split video and save to another file
--segment       #Split video into multiple video segments
--circular      #Write video to the circular buffer
--inline        #Write header in each I frame (only supports H.264)
--listen        #Wait for a TCP connection
--frames        #Set the number of frames recorded
For more camera setup instructions, please refer to Official Camera Documentation.
Working with Raspberry Pi 5 (libcamera)
Bookworm will not support "libcamera-*" anymore, so please replace it with "rpicam-*" as soon as possible. As shown below, the first command is for buster and bullseye systems and the second command is for bookworm systems.

sudo nano /boot/config.txt
sudo nano /boot/firmware/config.txt
Add the following code at the end of the above file:

dtoverlay=imx219,cam0
dtoverlay=imx219,cam1
Reboot it:

sudo reboot
Then open the terminal and execute:

libcamera-hello -t 0 --camera 0
libcamera-hello -t 0 --camera 1
Preface
To check what version of the system you are using, run sudo cat /etc/os-release to see if there is any information about the following two images, and then select.

The Raspberry Pi OS Bookworm changed the camera capture application from libcamera-* to rpicam-*, which currently allows users to continue using the old libcamera, but libcamera will be deprecated in the future, so please use rpicam as soon as possible.
If you are using the Raspberry Pi OS Bullseye system, please scroll down the page to use this tutorial's libcamera-* section.
RPicam
When running the latest version of Raspberry Pi OS, rpicam-apps already has five basic features installed. In this case, the official Raspberry Pi camera will also be detected and automatically enabled.
You can check if everything is working by inputting the following:

rpicam-hello 
You should see a camera preview window for about five seconds.
Note: If you're running on a Bullseye Raspberry Pi 3 and earlier, you need to re-enable Glamor to ensure that the X Windows hardware acceleration preview window works properly. Input the command sudo raspi-config in the terminal window, then select Advanced Options, Glamor, and Yes. Exit and restart your Raspberry Pi. By default, Raspberry Pi 3 and earlier devices running Bullseye may not be using the correct display driver. Please refer to the /boot/firmware/config.txt file and make sure that dtoverlay=vc4-fkms-v3d or dtoverlay=vc4-kms-v3d is currently active. If you need to change this setting, please restart.

rpicam-hello
It is equivalent to the camera's "hello world", it starts the camera preview stream and displays it on the screen, which can be stopped by clicking the window's Close button or by using ctrl^C in the terminal.
rpicam-hello -t 0
Tuning file
The libcamera for Raspberry Pi has tuning files for each type of camera module. The parameters in the file are passed to the algorithm and hardware to produce the best quality image. libcamera can only automatically determine the image sensor being used, not the entire module, even if the entire module affects the "tuning". As a result, it is sometimes necessary to override the default tuning file for a particular sensor.
For example, a sensor without an infrared filter (NoIR) version requires a different AWB (white balance) setting than the standard version, so an IMX219 NoIR used with a Pi 4 or earlier device should operate as follows:

rpicam-hello --tuning-file /usr/share/libcamera/ipa/rpi/vc4/imx219_noir.json
The Raspberry Pi 5 uses different tuning files in different folders, so here you will use:

rpicam-hello --tuning-file /usr/share/libcamera/ipa/rpi/pisp/imx219_noir.json
This also means that users can copy existing tuning files and modify them according to their preferences, as long as the parameter --tuning-file points to the new version.
The --tuning-file parameter is applicable to all rpicam-apps just like other command-line options.

rpicam-jpeg
rpicam-jpeg is a simple static image capture application.
To capture a full-resolution JPEG image, use the following command. This will display a preview for approximately five seconds, then capture the full-resolution JPEG image to the file test.jpg.

rpicam-jpeg -o test.jpg
The -t <duration> option is used to change the duration of the preview display, and the --width and --height options will alter the resolution of the captured static images. For example:

rpicam-jpeg -o test.jpg -t 2000 --width 640 --height 480
Exposure control
All of this rpicam-apps allow the user to run the camera with a fixed shutter speed and gain. Capture an image with an exposure time of 20ms and a gain of 1.5x. This gain will be used as an analog gain within the sensor until it reaches the maximum analog gain allowed by the core sensor driver. After that, the rest will be used as a digital gain

rpicam-jpeg -o test.jpg -t 2000 --shutter 20000 --gain 1.5
The AEC/AGC algorithm for the Raspberry Pi enables application-defined exposure compensation, allowing images to be made darker or brighter through a specified number of stops.

rpicam-jpeg --ev -0.5 -o darker.jpg
rpicam-jpeg --ev 0 -o normal.jpg
rpicam-jpeg --ev 0.5 -o brighter.jpg
Digital gain
The digital gain is applied by the ISP, not by the sensor. The digital gain will always be very close to 1.0 unless:

The total gain requested (via the option --gain or via the exposure profile in the camera adjustment) exceeds the gain that can be used as an analog gain within the sensor. Only the extra gain required will be used as a digital gain.
One of the color gains is less than 1 (note that the color gain is also used as a digital gain). In this case, the published digital gain will stabilize at 1 / min (red gain, blue gain). This means that one of the color channels (instead of the green channel) has a single-digit gain applied.
AEC/AGC is changing. When AEC/AGC moves, the digital gain typically changes to some extent in an attempt to eliminate any fluctuations, but it quickly returns to its normal value.
rpicam-still
It emulates many features of the original app raspistill.

rpicam-still -o test.jpg
Encoder
rpicam-still allows files to be saved in a number of different formats. It supports PNG and BMP encoding. It also allows the file to be saved as a binary dump of RGB or YUV pixels with no encoding or file format. In the latter case, the application that reads the file must be aware of its own pixel arrangement.

rpicam-still -e png -o test.png
rpicam-still -e bmp -o test.bmp
rpicam-still -e rgb -o test.data
rpicam-still -e yuv420 -o test.data
Note: The format in which the image is saved depends on the -e (equivalent to encoding) option and is not automatically selected based on the output file name.

Raw image capture
The raw image is an image that is produced directly by an image sensor, before any processing of it by the ISP (Image Signal Processor) or any CPU core. For color image sensors, these are usually Bayer format images. Note that the original image is very different from the processed but unencoded RGB or YUV images we saw before.
Get the raw image:

rpicam-still --raw --output test.jpg
Here, the -r option (also known as raw) indicates capturing the raw image and JPEG. In fact, the original image is the raw image that generates a JPEG. The original images are saved in DNG (Adobe Digital Negative) format and are compatible with many standard applications such as draw or RawTherapee. The original image is saved to a file with the same name but an extension of .ng, thus becoming test.dng.
These DNG files contain metadata related to image capture, including black level, white balance information, and the color matrix used by the ISP to generate JPEGs. This makes these DNG files more convenient to use for "manual" original conversion with some of the above tools in the future. Use exiftool to display all metadata encoded into a DNG file:

File Name                       : test.dng
Directory                       : .
File Size                       : 24 MB
File Modification Date/Time     : 2021:08:17 16:36:18+01:00
File Access Date/Time           : 2021:08:17 16:36:18+01:00
File Inode Change Date/Time     : 2021:08:17 16:36:18+01:00
File Permissions                : rw-r--r--
File Type                       : DNG
File Type Extension             : dng
MIME Type                       : image/x-adobe-dng
Exif Byte Order                 : Little-endian (Intel, II)
Make                            : Raspberry Pi
Camera Model Name               : /base/soc/i2c0mux/i2c@1/imx477@1a
Orientation                     : Horizontal (normal)
Software                        : rpicam-still
Subfile Type                    : Full-resolution Image
Image Width                     : 4056
Image Height                    : 3040
Bits Per Sample                 : 16
Compression                     : Uncompressed
Photometric Interpretation      : Color Filter Array
Samples Per Pixel               : 1
Planar Configuration            : Chunky
CFA Repeat Pattern Dim          : 2 2
CFA Pattern 2                   : 2 1 1 0
Black Level Repeat Dim          : 2 2
Black Level                     : 256 256 256 256
White Level                     : 4095
DNG Version                     : 1.1.0.0
DNG Backward Version            : 1.0.0.0
Unique Camera Model             : /base/soc/i2c0mux/i2c@1/imx477@1a
Color Matrix 1                  : 0.8545269369 -0.2382823821 -0.09044229197 -0.1890484985 1.063961506 0.1062747385 -0.01334283455 0.1440163847 0.2593136724
As Shot Neutral                 : 0.4754476844 1 0.413686484
Calibration Illuminant 1        : D65
Strip Offsets                   : 0
Strip Byte Counts               : 0
Exposure Time                   : 1/20
ISO                             : 400
CFA Pattern                     : [Blue,Green][Green,Red]
Image Size                      : 4056x3040
Megapixels                      : 12.3
Shutter Speed                   : 1/20
We have noticed that there is only one calibration light source (determined by the AWB algorithm, although it is always labeled as "D65"), and dividing the ISO number by 100 gives the analog gain being used.

Ultra-long exposure
In order to capture long-exposure images, disable AEC/AGC and AWB, as these algorithms will force the user to wait many frames while converging.
The way to disable them is to provide explicit values. Additionally, the --immediate option can be used to skip the entire preview phase that is captured.
Therefore, to perform an exposure capture of 100 seconds, use:

rpicam-still -o long_exposure.jpg --shutter 100000000 --gain 1 --awbgains 1,1 --immediate
For reference, the maximum exposure times of the three official Raspberry Pi cameras can be found in this table.

rpicam-vid
rpicam-vid can help us capture video on our Raspberry Pi device. Rpicam-vid displays a preview window and writes the encoded bitstream to the specified output. This will produce an unpacked video bitstream that is not packaged in any container format (such as an mp4 file).

rpicam-vid uses H.264 encoding
For example, the following command writes a 10-second video to a file named test.h264:

rpicam-vid -t 10s -o test.h264
You can use VLC and other video players to play the result files:

VLC test.h264
On the Raspberry Pi 5, you can output directly to the MP4 container format by specifying the MP4 file extension of the output file:

rpicam-vid -t 10s -o test.mp4
Encoder
rpicam-vid supports dynamic JPEG as well as uncompressed and unformatted YUV420:

rpicam-vid -t 10000 --codec mjpeg -o test.mjpeg
rpicam-vid -t 10000 --codec yuv420 -o test.data
The codec option determines the output format, not the extension of the output file.
The segment option splits the output file into segments-sized chunks (in milliseconds). By specifying extremely short segments (1 millisecond), this allows for the convenient decomposition of a moving JPEG stream into individual JPEG files. For example, the following command combines a 1 millisecond segment with a counter in the output filename to generate a new filename for each segment:

rpicam-vid -t 10000 --codec mjpeg --segment 1 -o test%05d.jpeg
Capture high frame rate video
To minimize frame loss for high frame rate (> 60fps) video, try the following configuration adjustments:

Set the target level of H.264 to 4.2 with the parameter --level 4.2
Disable software color denoising processing by setting the denoise option to cdn_off.
Disable the display window for nopreview to release additional CPU cycles.
Set force_turbo=1 in /boot/firmware/config.txt to ensure that the CPU clock is not throttled during video capture. For more information, see the force_turbo documentation.
Adjust the ISP output resolution parameter to --width 1280 --height 720 or lower to achieve the frame rate target.
On the Raspberry Pi 4, you can overclock the GPU to improve performance by adding a frequency of gpu_freq=550 or higher in /boot/firmware/config.txt. For detailed information, please refer to the Overclocking documentation.
The following command demonstrates how to implement a 1280×720 120fps video:

rpicam-vid --level 4.2 --framerate 120 --width 1280 --height 720 --save-pts timestamp.pts -o video.264 -t 10000 --denoise cdn_off -n
Integration of Libav with picam-vid
Rpicam-vid can encode audio and video streams using the ffmpeg/libav codec backend. You can save these streams to a file, or stream them over the network.
To enable the libav backend, pass libav to the codec option:

rpicam-vid --codec libav --libav-format avi --libav-audio --output example.avi
UDP
To use a Raspberry Pi as a server for streaming video over UDP, use the following command, replacing the < IP -addr> placeholder with the IP address of the client or multicast address, and replacing the <port> placeholder with the port you wish to use for streaming:

rpicam-vid -t 0 --inline -o udp://<ip-addr>:<port>
Use a Raspberry Pi as a client to view video streams over UDP, using the following command, replace the <port> placeholder with the port you want to stream:

vlc udp://@:<port> :demux=h264
Alternatively, use ffplay on the client side to stream with the following command:

ffplay udp://<ip-addr-of-server>:<port> -fflags nobuffer -flags low_delay -framedrop
TCP
Video can also be transmitted over TCP. Use Raspberry Pi as a server:

rpicam-vid -t 0 --inline --listen -o tcp://0.0.0.0:<port>
Use the Raspberry Pi as the client to view the video stream over TCP, use the following command:

vlc tcp/h264://<ip-addr-of-server>:<port>
Alternatively, use the ffplay stream at 30 frames per second on the client side with the following command:

ffplay tcp://<ip-addr-of-server>:<port> -vf "setpts=N/30" -fflags nobuffer -flags low_delay -framedrop
RTSP
To transfer video via RTSP using VLC, using the Raspberry Pi as the server, use the following command:

rpicam-vid -t 0 --inline -o - | cvlc stream:///dev/stdin --sout '#rtp{sdp=rtsp://:8554/stream1}' :demux=h264
To view the video stream on the RTSP using the Raspberry Pi as a client, use the following command:

ffplay rtsp://<ip-addr-of-server>:8554/stream1 -vf "setpts=N/30" -fflags nobuffer -flags low_delay -framedrop
Or on the client side use the following command to stream with VLC:

vlc rtsp://<ip-addr-of-server>:8554/stream1
If you need to close the preview window on the server, use the nopreview command.
Use inline flags to enforce stream header information into each inner frame, which helps the client understand the stream when the beginning is missed.

rpicam-raw
rpicam-raw records the video directly from the sensor as the original Bayer frame. It doesn't show the preview window. To record a two-second raw clip to a file called test.raw, run the following command:

rpicam-raw -t 2000 -o test.raw
RPICAM-RAW outputs raw frames without any format information. The application prints the pixel format and image size to the terminal window to help the user parse the pixel data.
By default, rpicam-raw outputs raw frames in a single, potentially very large file. Use the segment option to direct each raw frame to a separate file, using the %05d directive to make each frame filename unique:

rpicam-raw -t 2000 --segment 1 -o test%05d.raw
Using a fast storage device, rpicam-raw can write 18MB frames from a 12-megapixel HQ camera at a speed of 10fps to the disk. rpicam-raw is unable to format the output frame as a DNG file; To do this, use the rpicam-still option at a frame rate lower than 10 to avoid frame drops:

rpicam-raw -t 5000 --width 4056 --height 3040 -o test.raw --framerate 8
For more information on the original format, see mode documentation.

rpicam-detect
Note: The Raspberry Pi operating system does not include rpicam-detect. If you already have TensorFlow Lite installed, you can build rpicam-detect. For more information, see the instructions on building rpicam-apps in build. Don't forget to pass -DENABLE_TFLITE=1 when running cmake.
rpicam-detect displays a preview window and monitors the content using a Google MobileNet v1 SSD (Single Shot Detector) neural network that has been trained to recognize about 80 classes of objects using the Coco dataset. Rpicam-detect can recognize people, cars, cats and many other objects.
Whenever rpicam-detect detects a target object, it captures a full-resolution JPEG. Then return to monitoring preview mode.
For general information about model usage, please refer to the TensorFlow Lite Object Detector section. For example, when you are out, you can keep an eye on your cat:

rpicam-detect -t 0 -o cat%04d.jpg --lores-width 400 --lores-height 300 --post-process-file object_detect_tf.json --object cat
rpicam Parameter Settings
--help -h prints all the options, along with a brief description of each option
rpicam-hello -h
--version outputs the version strings for libcamera and rpicam-apps
rpicam-hello --version
Example output:

rpicam-apps build: ca559f46a97a 27-09-2021 (14:10:24)
libcamera build: v0.0.0+3058-c29143f7
--list-cameras lists the cameras connected to the Raspberry Pi and their available sensor modes
rpicam-hello --list-cameras
The identifier for the sensor mode has the following form:

S<Bayer order><Bit-depth>_<Optional packing> : <Resolution list>
Cropping is specified in the native sensor pixels (even in pixel binning mode) as (<x>, <y>)/<Width>×<Height>. (x, y) specifies the position of the width × height clipping window in the sensor array.
For example, the following output shows information for an IMX219 sensor with index 0 and an IMX477 sensor with index 1:

Available cameras
-----------------
0 : imx219 [3280x2464] (/base/soc/i2c0mux/i2c@1/imx219@10)
    Modes: 'SRGGB10_CSI2P' : 640x480 [206.65 fps - (1000, 752)/1280x960 crop]
                             1640x1232 [41.85 fps - (0, 0)/3280x2464 crop]
                             1920x1080 [47.57 fps - (680, 692)/1920x1080 crop]
                             3280x2464 [21.19 fps - (0, 0)/3280x2464 crop]
           'SRGGB8' : 640x480 [206.65 fps - (1000, 752)/1280x960 crop]
                      1640x1232 [41.85 fps - (0, 0)/3280x2464 crop]
                      1920x1080 [47.57 fps - (680, 692)/1920x1080 crop]
                      3280x2464 [21.19 fps - (0, 0)/3280x2464 crop]
1 : imx477 [4056x3040] (/base/soc/i2c0mux/i2c@1/imx477@1a)
    Modes: 'SRGGB10_CSI2P' : 1332x990 [120.05 fps - (696, 528)/2664x1980 crop]
           'SRGGB12_CSI2P' : 2028x1080 [50.03 fps - (0, 440)/4056x2160 crop]
                             2028x1520 [40.01 fps - (0, 0)/4056x3040 crop]
                             4056x3040 [10.00 fps - (0, 0)/4056x3040 crop]
--camera selects the camera you want to use. Specify an index from the list of available cameras.
rpicam-hello --list-cameras 0 
rpicam-hello --list-cameras 1 
--config -c specifies a file that contains the command parameter options and values. Typically, a file named example_configuration.txt specifies options and values as key-value pairs, with each option on a separate line.
timeout=99000
verbose=
Notice: Omission of prefixes -- for parameters is typically used in command lines. For flags that are missing values, such as verbose in the example above, a trailing = must be included.
Then you can run the following command to specify a timeout of 99,000 milliseconds and detailed output:

rpicam-hello --config example_configuration.txt 
--time -t, the default delay of 5000 milliseconds
rpicam-hello -t  
Specifies how long the application runs before shutting down. This applies to both the video recording and preview windows. When capturing a still image, the application displays a preview window with a timeout millisecond before outputting the captured image.

rpicam-hello -t 0
--preview sets the position (x,y coordinates) and size (w,h dimensions) of the desktop or DRM preview window. There is no impact on the resolution or aspect ratio of the image requested from the camera
Pass the preview window dimensions in the following comma-separated form: x,y,w,h

rpicam-hello --preview 100,100,500,500
--fullscreen -f forces the preview window to use the entire screen, with no borders or title bars. Scale the image to fit the entire screen. Values are not accepted.
rpicam-hello -f
--qt-preview uses the Qt preview window, which consumes more resources than other options, but supports X window forwarding. Not compatible with full-screen flags. Values are not accepted.
rpicam-hello --qt-preview
--nopreview makes the application not display a preview window. Values are not accepted.
rpicam-hello --nopreview
--info-text
Default values: "#%frame (%fps fps) exp %exp ag %ag dg %dg"
When running in a desktop environment, set the provided string as the title of the preview window. The following image metadata substitutions are supported:

Command	Description
%frame	Frame sequence number
%fps	Instantaneous frame rate
%exp	Shutter speed at which an image is captured, in ms
%ag	Image analog gain controlled by sensor chip
%dg	Image number gain controlled by ISP
%rg	Gain of the red component of each pixel point
%bg	Gain of the blue component of each pixel point
%focus	The corner point measure of the image, the larger the value, the clearer the image
%lp	Diopter of the current lens (distance in 1/meter)
%afstate	Autofocus status (idle, scanning, focused, failed)
rpicam-hello --info-test "Focus measure: %focus" 
--width
--height
Each parameter accepts a number that defines the size of the image displayed in the preview window in pixels.
For rpicam-still, rpicam-jpeg, and rpicam-vid, specify the output resolution.
For rpicam-raw, specify the original frame resolution. For a camera with a 2×2 bin readout mode, specify a resolution that is equal to or less than the bin mode to capture 2×2 bin original frames.
For rpicam-hello, there is no effect.
Record a 1080p video

rpicam-vid -o test.h264 --width 1920 --height 1080
Capture a JPEG at a resolution of 2028×1520. If used with an HQ camera, the 2×2 bin mode is used, so the original file (test. ng) contains 2028×1520 original Bayer images.

rpicam-still -r -o test.jpg --width 2028 --height 1520
--viewfinder-width
--viewfinder-height
Each parameter accepts a number that defines the size of the image displayed in the preview window in pixels. The size of the preview window is not affected, as the image is resized to fit. Captured still images or videos are not affected.

rpicam-still --viewfinder-width 1920 --viewfinder-height 1080
--mode allows the camera mode to be specified in the following colon-separated format: <width>:<height>:<bit-depth>:<packing> and if the provided values do not match exactly, the system will select the closest available option for the sensor. You can use the packed(P) or unpacked(U) packaging format, which affects the format of the stored video and static images, but does not affect the format of the frames transmitted to the preview window.
Bit-depth and packing are optional. By default, Bit-depth is 12, and Packing is set to P (packed).
For information on the bit depth, resolution, and packing options available for the sensors, please refer to list-cameras.
As shown below:

4056:3040:12:P - 4056×3040 resolution, 12 bits/pixel, packed.
1632:1224:10 - 1632×1224 resolution, 10 bits/pixel.
2592:1944:10:U - 2592×1944 resolution, 10 bits/pixel, unpacked.
3264:2448 - 3264×2448 resolution.
--viewfinder-mode is the same as the mode option, but it applies to the data passed to the preview window. For more information, see mode documentation.
--lores-width and --lores-height
Provide a second low-resolution image stream from the camera, scaled down to a specified size. Each accepts a number to define the dimensions of a low-resolution stream (in pixels). Available in preview and video modes. Static capture is not provided. For RPICAM-vid, disable additional color denoising processing. It is useful for image analysis combined with Image post-processing.

rpicam-hello --lores-width 224 --lores-height 224
--hflip flips the image horizontally. Values are not accepted.
rpicam-hello --hflip -t 0
--vflip flips the image vertically. Values are not accepted.
 rpicam-hello --vflip -t 0
--rotation rotates the image extracted from the sensor. Only values of 0 or 180 are accepted.
rpicam-hello  --rotation 0
--roi crops the image extracted from the entire sensor domain. Accepts four decimal values, ranged 0 to 1, in the following format: <x>,<y>,<w>,<h>. Each of these values represents the percentage of available width and height as a decimal between 0 and 1.
These values define the following proportions:
<x>: X coordinates to skip before extracting an image
<y>: Y coordinates to skip before extracting an image
<w>: image width to extract
<h>: image height to extract
The default is 0,0,1,1 (starting with the first X coordinate and the first Y coordinate, using 100% of the image width and 100% of the image height).
Examples:
rpicam-hello --roi 0.25, 0.25, 0.5, 0.5 selects half of the total number of pixels cropped from the center of the image (skips the first 25% of the X coordinates and the first 25% of the Y coordinates, uses 50% of the total width of the image and 50% of the total height of the image).
rpicam-hello --roi 0,0,0.25,0.25 selects a quarter of the total number of pixels cropped from the top left corner of the image (skips the first 0% of the X coordinate and the first 0% of the Y coordinate, uses 25% of the width of the image and 25% of the height of the image).

--HDR default: Off, run the camera in HDR mode. If no value is passed, it is assumed to be auto. Accept one of the following values:
off - disables HDR.
auto - enables HDR on supported devices. If available, use the sensor's built-in HDR mode. If the sensor does not have a built-in HDR mode, the onboard HDR mode is used (if available).
single-exp enables HDR on supported devices. If available, use the sensor's built-in HDR mode. If the sensor does not have a built-in HDR mode, the onboard HDR mode is used (if available).
rpicam-hello --hdr
Use the onboard HDR mode, if available, even if the sensor has a built-in HDR mode. If the onboard HDR mode is not available, HDR is disabled.
Raspberry Pi 5 and higher versions of devices have the onboard HDR mode.
To check the HDR mode built into the sensor, add this option to the list of cameras.

Camera Control Options
The following options control the image processing and algorithms that affect the image quality of the camera.

sharpness
Sets the image clarity. Values from the following ranges are accepted:

0.0 refers to not applying sharpening
Values greater than 0.0 but less than 1.0 apply less sharpening than the default value
1.0 applies the default sharpening amount
Values greater than 1.0 apply additional sharpening
rpicam-hello --sharpness 0.0
contrast
Specifies the image contrast. Values from the following ranges are accepted:

0.0 applies minimum contrast ratio
Values greater than 0.0 but less than 1.0 apply a contrast that is less than the default value
1.0 applies the default contrast ratio
Values greater than 1.0 apply additional contrast
rpicam-hello --contrast 0.0
brightness
Specifies the image brightness, which is added as an offset of all pixels in the output image. Values from the following ranges are accepted:

-1.0 refers to minimum brightness (black)
0.0 applies standard brightness
1.0 applies maximum brightness (white)
For more uses, refer to ev.

rpicam-hello --brightness 1.0
saturation
Specifies the image color saturation. Values from the following ranges are accepted:

0.0 applies minimum saturation (grayscale)
Values greater than 0.0 but less than 1.0 apply saturation less than the default value
1.0 applies the default saturation
Values greater than 1.0 apply additional saturation
rpicam-hello --saturation  0.6
ev
Specifies the exposure value (EV) compensation for the image. A numeric value is accepted that is passed along the following spectrum to the target value of the automatic exposure/gain control (AEC/AGC) processing algorithm:

-10.0 applies the minimum target value
0.0 applies standard target value
10.0 applies the maximum target value
rpicam-hello --ev  10.0
shutter
Specifies the exposure time using the shutter, measured in microseconds. When you use this option, the gain can still be varied. If the camera's frame rate is too high, it doesn't allow the specified exposure time (for example, with a frame rate of 1 fps and an exposure time of 10,000 microseconds), the sensor will use the maximum exposure time allowed by the frame rate.
For a list of minimum and maximum shutter times for official cameras, see camera hardware documentation. Values higher than the maximum will result in undefined behavior.

rpicam-hello --shutter 10000
gain
The effect of analoggain is the same as gain
Sets the combined analog and digital gain. When the sensor drive can provide the required gain, only analog gain is used. When the analog gain reaches its maximum, the ISP applies the digital gain. Accepts a numeric value.
For a list of analogue gain limits, for official cameras, see the camera hardware documentation.
Sometimes, digital gain can exceed 1.0 even when the analogue gain limit is not exceeded. This can occur in the following situations:
Either of the colour gains drops below 1.0, which will cause the digital gain to settle to 1.0/min(red_gain,blue_gain). This keeps the total digital gain applied to any colour channel above 1.0 to avoid discolouration artefacts.
Slight variances during Automatic Exposure/Gain Control (AEC/AGC) changes.

rpicam-hello --gain 0.8
metering default value: centre
Sets the metering mode of the Automatic Exposure/Gain Control (AEC/AGC) algorithm. Accepts the following values:

centre - centre weighted metering
spot - spot metering
average - average or whole frame metering
custom - custom metering mode defined in the camera tuning file
For more information on defining a custom metering mode, and adjusting region weights in existing metering modes, see the Raspberry Tuning guide for the Raspberry Pi cameras and libcamera.

rpicam-hello --metering centre
exposure
Sets the exposure profile. Changing the exposure profile should not affect the image exposure. Instead, different modes adjust gain settings to achieve the same net result. Accepts the following values:

sport: short exposure, larger gains
normal: normal exposure, normal gains
long: long exposure, smaller gains
You can edit exposure profiles using tuning files. For more information, see the Tuning guide for the Raspberry Pi cameras and libcamera.

rpicam-hello --exposure sport
awb
Sets the exposure profile. Changing the exposure profile should not affect the image exposure. Instead, different modes adjust gain settings to achieve the same final result. Accepts the following values: Available white balance modes:

Mode	Color temperature
auto	2500K ~ 8000K
incadescent	2500K ~ 3000K
tungsten	3000K ~3500K
fluorescent	4000K ~ 4700K
indoor	3000K ~ 5000K
daylight	5500K ~ 6500 K
cloudy	7000K ~ 8500K
custom	A custom range defined in the tuning file
These values are only approximate: values could vary according to the camera tuning.
No mode fully disables AWB. Instead, you can fix colour gains with awbgains.
For more information on AWB modes, including how to define a custom one, see the Tuning guide for the Raspberry Pi cameras and libcamera.

rpicam-hello --awb auto
awbgains
Sets a fixed red and blue gain value to be used instead of an Auto White Balance (AWB) algorithm. Set non-zero values to disable AWB. Accepts comma-separated numeric input in the following format: <red_gain>，<blue_gain>

rpicam-jpeg -o test.jpg --awbgains 1.5,2.0
denoise
Default value: auto
Sets the denoising mode. Accepts the following values:

auto: Enables standard spatial denoise. Uses extra-fast colour denoise for video, and high-quality colour denoise for images. Enables no extra colour denoise in the preview window.
off: Disables spatial and colour denoise.
cdn_off: Disables colour denoise.
cdn_fast: Uses fast colour denoise.
cdn_fast: Uses high-quality colour denoise. Not appropriate for video/viewfinder due to reduced throughput.
Even fast colour denoise can lower framerates. High quality colour denoise significantly lowers framerates.

rpicam-hello --denoise off
tuning-file
Specifies the camera tuning file. The tuning file allows you to control many aspects of image processing, including the Automatic Exposure/Gain Control (AEC/AGC), Auto White Balance (AWB), colour shading correction, colour processing, denoising and more. Accepts a tuning file path as input. For more information about tuning files, see Tuning Files.

autofocus-mode
Default value: default Specifies the autofocus mode. Accepts the following values:

default: puts the camera into continuous autofocus mode unless lens-position or autofocus-on-capture override the mode to manual
manual: does not move the lens at all unless manually configured with lens-position
auto: only moves the lens for an autofocus sweep when the camera starts or just before capture if autofocus-on-capture is also used
continuous: adjusts the lens position automatically as the scene changes
This option is only supported for certain camera modules.

rpicam-hello --autofocus-mode auto
autofocus-range
Default value: normal
Specifies the autofocus range. Accepts the following values:

normal: focuses from reasonably close to infinity
macro: focuses only on close objects, including the closest focal distances supported by the camera
full: focus on the entire range, from the very closest objects to infinity
This option is only supported for certain camera modules.

rpicam-hello autofocus-range normal
autofocus-speed
Default value: normal
Specifies the autofocus speed. Accepts the following values:

normal: changes the lens position at normal speed
fast: changes the lens position quickly
This option is only supported for certain camera modules.

rpicam-hello --autofocus-speed normal
autofocus-window
Specifies the autofocus window within the full field of the sensor. Accepts four decimal values, ranged 0 to 1, in the following format: <x>,<y>,<w>,<h>. Each of these values represents the percentage of available width and height as a decimal between 0 and 1.
These values define the following proportions:
<x>: X coordinates to skip before applying autofocus
<y>: Y coordinates to skip before applying autofocus
<w>:autofocus area width
<w>:autofocus area height
The default value uses the middle third of the output image in both dimensions (1/9 of the total image area).
Examples:

rpicam-hello—autofocus-window 0.25,0.25,0.5,0.5
selects exactly half of the total number of pixels cropped from the centre of the image (skips the first 25% of X coordinates, skips the first 25% of Y coordinates, uses 50% of the total image width, uses 50% of the total image height).

rpicam-hello—autofocus-window 0,0,0.25,0.25
selects exactly a quarter of the total number of pixels cropped from the top left of the image (skips the first 0% of X coordinates, skips the first 0% of Y coordinates, uses 25% of the image width, uses 25% of the image height).
This option is only supported for certain camera modules.

lens-position
Default value: default Moves the lens to a fixed focal distance, normally given in dioptres (units of 1 / distance in metres). Accepts the following spectrum of values:

0.0: moves the lens to the "infinity" position
Any other number: moves the lens to the 1 / number position. For example, the value 2.0 would focus at approximately 0.5m
normal: move the lens to a default position which corresponds to the hyperfocal position of the lens
Lens calibration is imperfect, so different camera modules of the same model may vary.

verbose
Alias: -v
Default value: 1 Sets the verbosity level. Accepts the following values:

0: no output
1: normal output
2: verbose output
rpicam-hello --verbose 1
For more details, click here for reference.

Resources
Demo codes
Demo codes
mpu9250 demo
Demo codes with calibration example
Drawing
Dimension
IMX210-83 Stereo Camera 3D Drawing
Related resources
Jetson Nano Forum
Nvidia Jetson Github
Jetson Nano Developer Kit
Getting started with Jetson Nano
NVIDIA Multimedia Documentation
Jetson Nano Developer Kit User Guide
Jetson Nano Developer Kit 3D Drawing
NVIDIA Official Free AI Tutorial (based on Jetson nano)
Datasheet
ICM20948 datasheet
FAQ
Question:Why wouldn't the camera world with Cheese?
 Answer:
The cheese tool could only work with the USB camera. The IMX219 series are CSI camera. Please use the provided test command.



Question:The camera could not work with opencv?
 Answer:
The CSI camera could work with the Gstreamer pipeline, please check the calling method.




Question:What is the operating temperature range of the IMX219 series?
 Answer:
0-60°.




Question:May I know what is the logic level of the icm20948 on the camera module?
 Answer:
It is 3.3V.




Question:Are the IMU data and camera data of this module synchronized above the timestamp?
 Answer:
Not synchronized.




Question:Is there a way to synchronize the frame from two cameras?
 Answer:
The IMX219-83 doesn't feature hardware synchronization and it is hard to synchronize, sorry about it.




Support



Technical Support

If you need technical support or have any feedback/review, please click the Submit Now button to submit a ticket, Our support team will check and reply to you within 1 to 2 working days. Please be patient as we make every effort to help you to resolve the issue.
Working Time: 9 AM - 6 PM GMT+8 (Monday to Friday)
