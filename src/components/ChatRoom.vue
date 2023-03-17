<template>
  <div class="column" v-if="leftRoom == room">
    <img
      src="@/assets/icons8-home-48.png"
      @click="returnHome"
      @contextmenu="returnHomeNewTab"
      @contextmenu.prevent
      class="home-button"
    /><img
      src="@/assets/icons8-update-left-rotation-50.png"
      @click="refreshPage"
      class="refresh-button"
    /><br /><br />
    You left the room. Refresh to rejoin.
  </div>
  <div v-else-if="userNotAllowedInRoom != room">
    <div class="column">
      <div>
        <img
          src="@/assets/icons8-home-48.png"
          @click="returnHome"
          @contextmenu="returnHomeNewTab"
          @contextmenu.prevent
          class="home-button"
        />
        <img
          v-if="!showMembers"
          src="@/assets/icons8-management-48.png"
          @click="showRoomMembers"
          @contextmenu.prevent
          class="show-members"
        />
        <img
          v-else
          src="@/assets/icons8-communication-50.png"
          @click="hideRoomMembers"
          @contextmenu.prevent
          class="show-chat"
        />
      </div>

      <label for="name">Group Name:</label><br /><br />
      <div v-if="!editDisplayName" class="inline">
        <img
          src="@/assets/icons8-edit-24.png"
          @click="edit"
          class="edit-button"
        /><strong>{{ displayName }}</strong>
      </div>
      <div v-else>
        <input
          id="name"
          type="text"
          maxlength="150"
          v-model="editableDisplayName"
          ref="editName"
        />
        <br /><img
          src="@/assets/icons8-checkmark-48.png"
          @click="updateDisplayName"
          class="edit-button"
        /><img
          src="@/assets/icons8-cancel-48.png"
          @click="cancelEdit"
          class="edit-button"
        />
      </div>
      <br />
      <div v-if="!showMembers">
        <div id="messages">
          <span
            v-for="notification in messageNotifications"
            :key="notification.id"
          >
            <br />
            <div
              :class="
                notification.read ? 'notification' : 'unread-notification'
              "
              @contextmenu.prevent
            >
              <strong>
                {{ notification.message__creator__display_name }}</strong
              >
              <br />
              <div>
                <audio
                  controls
                  :src="notification.url"
                  controlsList="nodownload nofullscreen noremoteplayback noplaybackrate"
                ></audio>
              </div>
              {{ notification.readable_timestamp }} <br />
            </div>
            <br />
          </span>
        </div>
        <br />
        <div class="record-playback" v-if="!isRecording">
          <div v-if="recordingData.length == 0">
            <img
              src="@/assets/icons8-microphone-60.png"
              @click="addRecording"
              @contextmenu.prevent
              class="add-record"
            />
          </div>
          <div class="playback" v-else>
            <img
              v-if="shareable"
              src="@/assets/icons8-send-48.png"
              @click="send"
              @contextmenu.prevent
              class="send-button"
            />
            <img
              src="@/assets/icons8-microphone-60.png"
              @click="recordAudio"
              @contextmenu.prevent
              class="record-button"
            />
            <audio
              controls
              :src="recordedAudioUrl"
              controlsList="nodownload nofullscreen noremoteplayback noplaybackrate"
            ></audio>
            <img
              src="@/assets/icons8-bin-48.png"
              @click="deleteRecorded"
              @contextmenu.prevent
              class="bin-button"
            />
          </div>
        </div>
        <div class="recording" v-else>
          <div>
            <img
              src="@/assets/icons8-pause-squared-48.png"
              @click="pauseRecording"
              @contextmenu.prevent
              class="pause-button"
            />
          </div>
        </div>
      </div>
      <div v-else>
        <div id="members">
          <b>Group members:</b><br /><br />
          <span v-for="member in roomMembers" :key="member">
            {{ member }}<br />
          </span>
        </div>
        <img
          v-if="shareable"
          src="@/assets/icons8-add-users-48.png"
          @click="share"
          @contextmenu.prevent
          class="share-button"
        /><br /><br />
        <Toggle v-model="privateRoom" @change="updatePrivacy">
          <template v-slot:label="{ checked, classList }">
            <span :class="classList.label">{{
              checked ? "Private" : "Public"
            }}</span>
          </template>
        </Toggle>
        <br /><br />
        <div id="join-requests" v-if="privateRoom">
          <span><b>Users requesting to join:</b><br /><br /></span>
          <div v-if="joinRequests.length > 0" id="requests">
            <span v-for="request in joinRequests" :key="request.user">
              {{ request.user__display_name }}
              <div class="btn-group">
                <button
                  type="button"
                  class="btn"
                  @click="acceptRequest(request.user__username)"
                >
                  Accept
                </button>
                <div class="divider" />
                <button
                  type="submit"
                  class="btn btn__primary"
                  @click="rejectRequest(request.user__username)"
                >
                  Reject
                </button>
              </div>
              <br
            /></span>
          </div>
          <div v-else>None</div>
          <br />
        </div>
      </div>
    </div>
  </div>
  <div class="column" v-else>
    <img
      src="@/assets/icons8-home-48.png"
      @click="returnHome"
      @contextmenu="returnHomeNewTab"
      @contextmenu.prevent
      class="home-button"
    />
    <br />
    You are not allowed in private room. Access requested.
  </div>
</template>

<script>
import Toggle from "@vueform/toggle";
export default {
  name: "ChatRoom",
  components: {
    Toggle,
  },
  props: {
    room: {
      type: String,
      required: true,
    },
    authToken: {
      type: String,
      required: true,
    },
    userId: {
      type: String,
      required: true,
    },
    roomWebSocket: {
      type: WebSocket,
      required: false,
    },
    roomMembers: {
      type: Array,
      required: true,
    },
    privacy: {
      type: Boolean,
      required: true,
    },
    userNotAllowedInRoom: {
      type: String,
      required: true,
    },
    joinRequests: {
      type: Array,
      required: true,
    },
    leftRoom: {
      type: String,
      required: true,
    },
    displayName: {
      type: String,
      required: true,
    },
    uploadUrl: {
      type: String,
      required: true,
    },
    messageNotifications: {
      type: Array,
      required: true,
    },
  },
  data() {
    return {
      showMembers: false,
      shareable: null,
      privateRoom: false,
      editDisplayName: false,
      editableDisplayName: null,
      audio: null,
      isRecording: false,
      recorder: null,
      recordingData: [],
      recordedAudioUrl: null,
      recordingFile: null,
    };
  },
  methods: {
    showRoomMembers: function () {
      this.showMembers = true;
      if (this.isRecording) {
        this.pauseRecording();
      }
    },
    hideRoomMembers: function () {
      this.showMembers = false;
    },
    send: function () {
      const requestOptions = {
        method: "PUT",
        headers: { "Content-Type": "audio/ogg" },
        body: this.recordingFile,
      };
      fetch(this.uploadUrl, requestOptions)
        .then(() => {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "send_message",
            })
          );
          this.deleteRecorded();
        })
        .catch((error) => console.log(error));
    },
    pauseRecording: function () {
      if (this.recorder) {
        this.recorder.pause();
        this.recordingFile = new Blob(this.recordingData, {
          type: "audio/ogg; codecs=opus",
        });
        if (this.recordedAudioUrl) {
          window.URL.revokeObjectURL(this.recordedAudioUrl);
        }
        this.recordedAudioUrl = window.URL.createObjectURL(this.recordingFile);
      }
    },
    addRecording: function () {
      navigator.permissions.query({ name: "microphone" }).then((permission) => {
        if (permission.state === "granted") {
          if (!this.audio) {
            this.audio = navigator.mediaDevices.getUserMedia({ audio: true });
          }
          this.recordAudio();
        } else {
          this.audio = navigator.mediaDevices.getUserMedia({ audio: true });
        }
      });
    },
    recordAudio: function () {
      if (!this.recorder || this.recorder.state == "inactive") {
        this.audio.then((stream) => {
          this.recorder = new MediaRecorder(stream);
          this.recorder.onstart = () => {
            this.isRecording = true;
          };
          this.recorder.onresume = () => {
            this.isRecording = true;
          };
          this.recorder.onpause = () => {
            this.isRecording = false;
          };
          this.recorder.ondataavailable = (event) => {
            this.recordingData.push(event.data);
          };
          this.recorder.start(0); //0 for as little audio buffering as possible so recording starts immediately
        });
      } else {
        this.recorder.resume();
      }
    },
    deleteRecorded: function () {
      this.recorder.ondataavailable = () => {};
      this.recorder.stop();
      this.recordingFile = null;
      this.recordingData = [];
    },
    returnHome: function () {
      const url = new URL(window.location.href);
      window.history.replaceState("", "", url.origin);
      this.$emit("go-home");
      this.roomWebSocket.send(
        JSON.stringify({
          command: "disconnect",
        })
      );
    },
    returnHomeNewTab: function () {
      const url = new URL(window.location.href);
      window.open(url.origin, "_blank");
    },
    refreshPage: function () {
      location.reload();
    },
    updatePrivacy: function () {
      if (!this.privateRoom) {
        this.roomWebSocket.send(
          JSON.stringify({ command: "approve_all_users" })
        );
      }
      this.roomWebSocket.send(
        JSON.stringify({
          command: "update_privacy",
          privacy: this.privateRoom,
        })
      );
    },
    acceptRequest: function (username) {
      this.roomWebSocket.send(
        JSON.stringify({ command: "approve_user", username: username })
      );
    },
    rejectRequest: function (username) {
      this.roomWebSocket.send(
        JSON.stringify({ command: "reject_user", username: username })
      );
    },
    share: function () {
      const shareData = {
        url: window.location.href,
      };
      navigator.share(shareData);
    },
    edit: function () {
      this.editDisplayName = true;
      this.editableDisplayName = this.displayName;
      this.$nextTick(() => {
        this.$refs.editName.select();
      });
    },
    cancelEdit: function () {
      this.editDisplayName = false;
      this.editableDisplayName = this.displayName;
    },
    updateDisplayName: function () {
      this.editDisplayName = false;
      this.roomWebSocket.send(
        JSON.stringify({
          command: "update_display_name",
          name: this.editableDisplayName,
        })
      );
    },
  },
  watch: {
    privacy(newPrivacy) {
      this.privateRoom = newPrivacy;
    },
  },
  created() {
    this.shareable =
      typeof navigator.share === "function" &&
      navigator.canShare({
        url: window.location.href,
      });
    this.privateRoom = this.privacy;
  },
};
</script>

<style scoped>
.notification {
  padding: 6px 10px;
  border-radius: 50%;
  border-style: solid;
  border-color: #10b981;
  overflow: clip;
}
.unread-notification {
  padding: 6px 10px;
  border-radius: 50%;
  border-style: solid;
  border-color: #10b981;
  background-color: rgb(76, 178, 247);
  overflow: clip;
}
.show-members {
  padding: 6px 10px;
  border-radius: 70%;
  cursor: pointer;
}
.show-members:hover {
  background: #e0e0e0;
}
.show-chat {
  padding: 6px 10px;
  border-radius: 70%;
  cursor: pointer;
}
.show-chat:hover {
  background: #e0e0e0;
}
.send-button {
  cursor: pointer;
  transition: 0.2s;
}
.send-button:hover {
  transform: scale(1.1);
}
.add-record {
  padding: 6px 10px;
  cursor: pointer;
  transition: 0.2s;
}
.add-record:hover {
  transform: scale(1.1);
}
.recording {
  display: flex;
  justify-content: center;
}
.record-button {
  cursor: pointer;
  transition: 0.2s;
}
.record-button:hover {
  transform: scale(1.1);
}
.record-playback {
  display: flex;
  justify-content: center;
}
.playback {
  transform: scale(0.9);
  display: flex;
  justify-content: center;
}
.pause-button {
  cursor: pointer;
  transition: 0.2s;
}
.pause-button:hover {
  transform: scale(1.1);
}
.bin-button {
  cursor: pointer;
  transition: 0.2s;
}
.bin-button:hover {
  transform: scale(1.1);
}
.inline {
  display: flex;
  align-items: center;
  justify-content: center;
}
.divider {
  width: 50px;
  height: auto;
  display: inline-block;
}
#members {
  word-break: break-word;
  max-height: 400px;
  overflow-y: scroll;
}
/* Hide scrollbar for Chrome, Safari and Opera */
#members::-webkit-scrollbar {
  display: none;
}

/* Hide scrollbar for IE, Edge and Firefox */
#members {
  -ms-overflow-style: none; /* IE and Edge */
  scrollbar-width: none; /* Firefox */
}
#join-requests {
  word-break: break-word;
  max-height: 400px;
  overflow-y: scroll;
}
/* Hide scrollbar for Chrome, Safari and Opera */
#join-requests::-webkit-scrollbar {
  display: none;
}

/* Hide scrollbar for IE, Edge and Firefox */
#join-requests {
  -ms-overflow-style: none; /* IE and Edge */
  scrollbar-width: none; /* Firefox */
}
#messages {
  max-height: 650px;
  overflow-y: scroll;
}
/* Hide scrollbar for Chrome, Safari and Opera */
#messages::-webkit-scrollbar {
  display: none;
}

/* Hide scrollbar for IE, Edge and Firefox */
#messages {
  -ms-overflow-style: none; /* IE and Edge */
  scrollbar-width: none; /* Firefox */
}
@media (orientation: landscape) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
@media (orientation: portrait) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
.home-button {
  padding: 6px 10px;
  border-radius: 50%;
  cursor: pointer;
}
.home-button:hover {
  background: #e0e0e0;
}
.refresh-button {
  padding: 6px 10px;
  border-radius: 50%;
  cursor: pointer;
}
.refresh-button:hover {
  background: #e0e0e0;
}
.share-button {
  padding: 6px 10px;
  border-radius: 70%;
  cursor: pointer;
}
.share-button:hover {
  background: #e0e0e0;
}
.btn:hover {
  background: #e0e0e0;
}
.edit-button {
  padding: 6px 10px;
  border-radius: 50%;
  cursor: pointer;
}
.edit-button:hover {
  background: #e0e0e0;
}

.name {
  padding-right: 8px;
  font-size: 11px;
}
</style>