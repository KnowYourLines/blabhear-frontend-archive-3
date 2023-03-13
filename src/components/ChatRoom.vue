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
          v-if="showMembers"
          src="@/assets/icons8-communication-50.png"
          @click="hideRoomMembers"
          @contextmenu.prevent
          class="show-chat"
        />
      </div>

      <label for="name">Group Name:</label><br /><br />
      <div v-if="!editDisplayName">
        <strong>{{ displayName }}</strong
        ><br /><img
          src="@/assets/icons8-edit-24.png"
          @click="edit"
          class="edit-button"
        />
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
      <div v-if="showMembers" id="room-members">
        <br />
        <Toggle v-model="privateRoom" @change="updatePrivacy">
          <template v-slot:label="{ checked, classList }">
            <span :class="classList.label">{{
              checked ? "Private" : "Public"
            }}</span>
          </template>
        </Toggle>
        <br /><br />
        <div v-if="privateRoom">
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
        <img
          v-if="shareable"
          src="@/assets/icons8-add-users-48.png"
          @click="share"
          @contextmenu.prevent
          class="share-button"
        />
        <div id="members">
          <b>Group members:</b><br /><br />
          <span v-for="member in roomMembers" :key="member">
            {{ member }}<br />
          </span>
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
  },
  data() {
    return {
      shareable: null,
      privateRoom: false,
      editDisplayName: false,
      editableDisplayName: null,
      messageToSend: "",
      showMembers: false,
    };
  },
  methods: {
    showRoomMembers: function () {
      this.showMembers = true;
      this.showRecordingSettings = false;
      if (this.isRecording) {
        this.pauseRecording();
      }
    },
    hideRoomMembers: function () {
      this.showMembers = false;
      this.showRecordingSettings = false;
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
    this.shareable = typeof navigator.share === "function";
    this.privateRoom = this.privacy;
  },
};
</script>

<style scoped>
.divider {
  width: 50px;
  height: auto;
  display: inline-block;
}
#members {
  word-break: break-word;
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