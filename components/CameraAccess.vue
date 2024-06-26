<template>
  <div class="media-capture">
    <video ref="video" autoplay></video>
    <div v-if="!cameraOpened" class="controls">
      <button @click="openCamera" class="btn open-camera">Open Camera</button>
    </div>
    <div v-else class="controls">
      <button @click="startRecording" class="btn start" :disabled="!canRecord">Start Recording</button>
      <button @click="stopRecording" class="btn stop" :disabled="!recording">Stop Recording</button>
      <button @click="captureImage" class="btn capture">Capture Image</button>
      <button @click="closeCamera" class="btn close">Close Camera</button>
      <button @click="switchCamera" class="btn switch-camera">Switch Camera</button>
    </div>
    <div v-if="mediaUrl" class="media-output">
      <h3>Captured Media</h3>
      <video v-if="isVideo" :src="mediaUrl" controls></video>
      <img v-else :src="mediaUrl" alt="Captured Image">
      <a :href="mediaUrl" :download="fileName" class="btn download">Download {{ isVideo ? 'Video' : 'Image' }}</a>
      <button @click="cancelMedia" class="btn cancel">Cancel</button>
    </div>
    <div v-if="!canRecord" class="unsupported">
      <p>Recording is not supported on this device/browser.</p>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      mediaRecorder: null,
      chunks: [],
      mediaUrl: null,
      isVideo: false,
      fileName: '',
      cameraOpened: false,
      recording: false,
      canRecord: 'MediaRecorder' in window,
      devices: [],
      currentDeviceId: null,
      backCameraDeviceId: null,
    };
  },
  methods: {
    async openCamera() {
      this.cameraOpened = true;
      await this.enumerateDevices();
      await this.startCamera();
    },
    async enumerateDevices() {
      const devices = await navigator.mediaDevices.enumerateDevices();
      this.devices = devices.filter(device => device.kind === 'videoinput');
      this.currentDeviceId = this.devices[0].deviceId;
      this.backCameraDeviceId = this.devices.find(device => device.label.toLowerCase().includes('back'))?.deviceId || this.devices[0].deviceId;
    },
    async startCamera(deviceId = this.currentDeviceId) {
      const stream = await navigator.mediaDevices.getUserMedia({ video: { deviceId }, audio: true });
      this.$refs.video.srcObject = stream;
    },
    async switchCamera() {
      this.currentDeviceId = this.currentDeviceId === this.backCameraDeviceId ? this.devices[0].deviceId : this.backCameraDeviceId;
      await this.startCamera(this.currentDeviceId);
    },
    async startRecording() {
      if (!this.canRecord) return;
      
      this.resetMedia();
      this.isVideo = true;
      this.fileName = 'video.webm';
      const stream = this.$refs.video.srcObject;
      this.mediaRecorder = new MediaRecorder(stream);
      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          this.chunks.push(event.data);
        }
      };
      this.mediaRecorder.onstop = () => {
        const blob = new Blob(this.chunks, { type: 'video/webm' });
        this.chunks = [];
        this.mediaUrl = URL.createObjectURL(blob);
      };
      this.mediaRecorder.start();
      this.recording = true;
    },
    stopRecording() {
      if (this.mediaRecorder) {
        this.mediaRecorder.stop();
        const stream = this.$refs.video.srcObject;
        const tracks = stream.getTracks();
        tracks.forEach((track) => track.stop());
        this.$refs.video.srcObject = null;
        this.recording = false;
      }
    },
    async captureImage() {
      this.resetMedia();
      this.isVideo = false;
      this.fileName = 'image.png';
      const video = this.$refs.video;
      const canvas = document.createElement('canvas');
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      const context = canvas.getContext('2d');
      context.drawImage(video, 0, 0, canvas.width, canvas.height);
      this.mediaUrl = canvas.toDataURL('image/png');
    },
    closeCamera() {
      this.cameraOpened = false;
      const stream = this.$refs.video.srcObject;
      const tracks = stream.getTracks();
      tracks.forEach((track) => track.stop());
      this.$refs.video.srcObject = null;
    },
    cancelMedia() {
      this.resetMedia();
    },
    resetMedia() {
      this.mediaUrl = null;
      this.isVideo = false;
      this.fileName = '';
    },
  },
};
</script>

<style scoped>
.media-capture {
  text-align: center;
  max-width: 600px;
  margin: auto;
}

video, img {
  max-width: 100%;
  margin: 10px 0;
}

.controls {
  margin: 20px 0;
}

.btn {
  background-color: #008cba; /* Blue background */
  border: none; /* Remove borders */
  color: white; /* White text */
  padding: 15px 32px; /* Some padding */
  text-align: center; /* Center the text */
  text-decoration: none; /* Remove underline */
  display: inline-block; /* Make the buttons appear beside each other */
  font-size: 16px; /* Increase font size */
  margin: 4px 2px; /* Add some margin */
  cursor: pointer; /* Add a pointer cursor on hover */
  border-radius: 12px; /* Rounded corners */
  transition: background-color 0.3s; /* Smooth transition for background color */
}

.btn:hover {
  background-color: #005f5f; /* Darker blue on hover */
}

.open-camera {
  background-color: #007bff; /* Blue background */
}

.open-camera:hover {
  background-color: #0056b3; /* Darker blue on hover */
}

.start {
  background-color: #4CAF50; /* Green background */
}

.start:hover {
  background-color: #45a049; /* Darker green on hover */
}

.stop {
  background-color: #f44336; /* Red background */
}

.stop:hover {
  background-color: #da190b; /* Darker red on hover */
}

.capture {
  background-color: #ff9800; /* Orange background */
}

.capture:hover {
  background-color: #e68900; /* Darker orange on hover */
}

.download {
  background-color: #007bff; /* Blue background */
  text-decoration: none; /* Remove underline */
}

.download:hover {
  background-color: #0056b3; /* Darker blue on hover */
}

.cancel {
  background-color: #6c757d; /* Gray background */
}

.cancel:hover {
  background-color: #5a6268; /* Darker gray on hover */
}

.close {
  background-color: #ff0000; /* Red background */
}

.close:hover {
  background-color: #cc0000; /* Darker red on hover */
}

.switch-camera {
  background-color: #00bcd4; /* Light Blue background */
}

.switch-camera:hover {
  background-color: #0097a7; /* Darker Light Blue on hover */
}

.media-output {
  margin-top: 20px;
}

.unsupported {
  margin-top: 20px;
  color: red;
}
</style>
