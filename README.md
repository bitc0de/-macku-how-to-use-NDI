All the necessary elements to download are in the `sources.zip` file to ensure you're using the same versions. However, you can also download them from the official repositories linked below.

The OBS plugin to use NDI is called **DistroAV**. We'll refer to it as such from now on.

## OBS

First, download OBS if you don't already have it.  
We'll use the latest version of OBS (31.0.2):  
👉 [https://obsproject.com/es/download](https://obsproject.com/es/download)

Install it like any other program.

## NDI Runtime v6+

To use NDI 6 and DistroAV, you need the new NDI Runtime. It improves latency, API connections, compatibility, and stability.

Download the NDI Runtime v6+ here:  
👉 [https://downloads.ndi.tv/SDK/NDI_SDK/NDI%206%20Runtime.exe](https://downloads.ndi.tv/SDK/NDI_SDK/NDI%206%20Runtime.exe)

Install it as you would any other application.

## DistroAV 6 (OBS Plugin)

Now download the OBS plugin that handles the NDI connection:  
👉 [https://github.com/DistroAV/DistroAV/releases/download/6.0.0/distroav-6.0.0-windows-x64-Installer.exe](https://github.com/DistroAV/DistroAV/releases/download/6.0.0/distroav-6.0.0-windows-x64-Installer.exe)

Once installed, open OBS, go to **Tools**, and you'll see **DistroAV Configuration**.

![DistroAV in Tools menu](https://i.ibb.co/BVds5k0J/image.png)

Open the configuration and change only the output name. Set it to `OBS`.

![Set output name](https://i.ibb.co/ymKKcpJ/image.png)

Now NDI is configured in OBS. No need to press Record — just having a scene and a source is enough.

![OBS Scene and Source](https://i.ibb.co/WN8bwvV3/image.png)

Restart OBS to correctly save the configuration and start NDI.

## GETTING THE IMAGE FROM ANOTHER PC ON THE SAME LOCAL NETWORK

To retrieve the image in real-time, we'll use a Python script.  
Make sure to install the `pyindi` dependency (I'm using version 0.0.1).

Here is the commented code:

```python
# Initialize NDI
# ndi.initialize() initializes the NDI library. If it fails, the program exits.
if not ndi.initialize():
    print("Failed to initialize NDI")
    exit(1)

# Create an NDI source finder
finder = ndi.find_create_v2()
# Wait up to 5 seconds to find available NDI sources
ndi.find_wait_for_sources(finder, 5000)
# Get the list of found NDI sources
sources = ndi.find_get_current_sources(finder)

# If no sources were found, exit
if not sources:
    print("No NDI sources found.")
    exit(1)

# Select the first available NDI source
source = sources[0]
# Configure NDI receiver settings
recv_settings = ndi.RecvCreateV3()
# Use the fastest color format
recv_settings.color_format = ndi.RECV_COLOR_FORMAT_FASTEST
# Use the lowest bandwidth to minimize latency
recv_settings.bandwidth = ndi.RECV_BANDWIDTH_LOWEST
# Create the NDI receiver
recv = ndi.recv_create_v3(recv_settings)

# Connect the receiver to the selected source
ndi.recv_connect(recv, source)
print(f"Connected to: {source.ndi_name}")

while True:
    # Capture a video frame from the NDI source
    frame_type, video_frame, _, _ = ndi.recv_capture_v2(recv, 1000)

    if frame_type != ndi.FRAME_TYPE_VIDEO or video_frame is None:
        continue

    # Convert the frame data from bytes to a numpy array
    frame_yuv = np.frombuffer(video_frame.data, dtype=np.uint8).reshape(
        (video_frame.yres, video_frame.xres, 2)
    )
    # Convert YUV to BGR (OpenCV format)
    frame_bgr_fullres = cv2.cvtColor(frame_yuv, cv2.COLOR_YUV2BGR_UYVY)
    # Make a copy for processing
    frame_resized = frame_bgr_fullres.copy()

# Cleanup on close
ndi.recv_destroy(recv)
ndi.find_destroy(finder)
ndi.destroy()
