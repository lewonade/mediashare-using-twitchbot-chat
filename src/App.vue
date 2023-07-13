<template>
  <div id="app" :class="{ 'is-invisible': !isPlaying }">
    <div ref="playerDiv"></div>
    <div class="loading-bar" v-if="isPlaying">
      <div class="progress" :style="{ width: progressBarWidth }"></div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, computed } from "vue";
import tmi from "tmi.js";

export default {
  name: "App",
  setup() {
    const playerDiv = ref(null);
    const videoQueue = ref([]);
    let player;
    let currentVideo = null;
    let isVideoLoaded = false;
    const isPlaying = ref(false);

    const playNextVideo = () => {
      if (videoQueue.value.length > 0 && player && !currentVideo) {
        const next = videoQueue.value[0]; // Get the next video in the queue without removing it

        player.loadVideoById({
          videoId: next.videoId,
          startSeconds: next.startTimeSeconds,
          endSeconds: next.endTimeSeconds,
        });

        currentVideo = next;
        isVideoLoaded = true;
      }
    };

    const onPlayerStateChange = (event) => {
      if (event.data === window.YT.PlayerState.ENDED) {
        currentVideo = null;
        isVideoLoaded = false;
        isPlaying.value = false;
        videoQueue.value.shift(); // Remove the completed video from the queue
        playNextVideo();
      } else if (event.data === window.YT.PlayerState.PLAYING) {
        isPlaying.value = true;
      }
    };

    const handleKeyboardShortcuts = (event) => {
      if (event.code === "F9") {
        if (
          player &&
          player.getPlayerState() === window.YT.PlayerState.PLAYING
        ) {
          player.pauseVideo();
        } else if (
          player &&
          player.getPlayerState() === window.YT.PlayerState.PAUSED
        ) {
          player.playVideo();
        }
      } else if (event.code === "F10") {
        if (!currentVideo && !isVideoLoaded && videoQueue.value.length > 0) {
          playNextVideo();
        }
      }
    };

    onMounted(() => {
      window.onYouTubeIframeAPIReady = () => {
        player = new window.YT.Player(playerDiv.value, {
          height: "390",
          width: "640",
          events: {
            onStateChange: onPlayerStateChange,
          },
        });
      };

      const options = {
        identity: {
          // you need to go over to this link https://dev.twitch.tv/console/apps and create an application on the URL- "http://localhost" with organization 'none' after that you take the client id and replace the "YOUR-CLIENT-ID" placeholder
          username: "YOUR-CLIENT-ID",

          // for this part you need to got to this url https://twitchapps.com/tmi/ and connect your twitch account, it will give you an oauth token for this you need to replace the "YOUR-OAUTH" with your oauth
          // it should look likeP this   -   password: 'oauth:qsdjvghd2vhd8f77duicbnsjk3',
          password: "YOUR-OAUTH",
        },
        channels: ["PUT-YOUR-TWITCH-NAME-HERE"],
      };

      const client = new tmi.Client(options);

      client.connect();

      client.on("message", (channel, tags, message) => {
        // Check for pause, start, skip, and return commands from youre Username
        if (tags.username.toLowerCase() === "PUT-YOUR-USERNAME-HERE") {
          if (
            message.trim().toLowerCase() === "!pause" &&
            player.getPlayerState() === window.YT.PlayerState.PLAYING
          ) {
            player.pauseVideo();
            return;
          } else if (
            message.trim().toLowerCase() === "!start" &&
            player.getPlayerState() === window.YT.PlayerState.PAUSED
          ) {
            player.playVideo();
            return;
          } else if (
            message.trim().toLowerCase() === "!skip" &&
            player.getPlayerState() !== window.YT.PlayerState.ENDED
          ) {
            player.stopVideo();
            currentVideo = null;
            isVideoLoaded = false;
            isPlaying.value = false;
            videoQueue.value.shift();
            playNextVideo();
            return;
          } else if (
            message.trim().toLowerCase() === "!return" &&
            player.getPlayerState() !== window.YT.PlayerState.ENDED
          ) {
            player.stopVideo();
            currentVideo = null;
            isVideoLoaded = false;
            isPlaying.value = false;
            videoQueue.value.unshift(videoQueue.value.shift());
            playNextVideo();
            return;
          }
        }

        const pattern =
          /(\d{2}:\d{2}-\d{2}:\d{2})?\s?(https:\/\/(www\.|m\.)?youtube\.com\/(watch\?v=|shorts\/)?[A-Za-z0-9_-]+|https:\/\/youtu\.be\/[A-Za-z0-9_-]+)/;

        if (pattern.test(message)) {
          const [, time, url] = message.match(pattern);
          const [startTime, endTime] = (time || "00:00-00:00").split("-");

          // Extract video ID from the URL depending on the format
          let videoId;
          if (url.includes("youtu.be")) {
            videoId = url.split("/")[3];
          } else if (url.includes("watch?v=")) {
            videoId = url.split("=")[1];
          } else if (url.includes("shorts/")) {
            videoId = url.split("/")[4];
          }

          const [startMinutes, startSeconds] = startTime.split(":");
          const [endMinutes, endSeconds] = endTime.split(":");

          const startTimeSeconds =
            Number(startMinutes) * 60 + Number(startSeconds);
          const endTimeSeconds = Number(endMinutes) * 60 + Number(endSeconds);

          videoQueue.value.push({
            videoId,
            startTimeSeconds,
            endTimeSeconds,
          });

          if (!currentVideo && !isVideoLoaded) {
            playNextVideo();
          }
        }
      });

      window.addEventListener("keydown", handleKeyboardShortcuts);
    });

    const progressBarWidth = computed(() => {
      if (player && currentVideo) {
        const currentTime = player.getCurrentTime();
        const videoDuration =
          currentVideo.endTimeSeconds - currentVideo.startTimeSeconds;
        const progressPercentage =
          (currentTime - currentVideo.startTimeSeconds) / videoDuration;
        return `${progressPercentage * 100}%`;
      }
      return "0%";
    });

    return {
      playerDiv,
      isPlaying,
      progressBarWidth,
    };
  },
};
</script>

<style>
.is-invisible {
  display: none;
}

.loading-bar {
  width: 100%;
  height: 10px;
  background-color: #ccc;
  margin-top: 10px;
}

.progress {
  height: 100%;
  background-color: #ff0000;
  width: 0;
  transition: width 1s linear;
}
</style>
