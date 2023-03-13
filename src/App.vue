<template>
  <div v-if="!room">
    <div v-if="isAnonymous">
      <div v-if="!verifyPhone">
        <img
          src="@/assets/icons8-decision-50.png"
          @click="verify"
          class="verify-button"
        />
      </div>
      <div v-else>
        <img
          src="@/assets/icons8-close-30.png"
          @click="cancelVerify"
          class="verify-button"
        />
        <PhoneSignIn @signed-in="signedIn" /><br />
      </div>
    </div>
    <div v-else>
      <img src="@/assets/icons8-checked-user-52.png" />
    </div>
    <HomePage
      :authToken="authToken"
      :userId="userId"
      :displayName="displayName"
      :notifications="notifications"
      :userWebSocket="userWebSocket"
      @new-room="newRoom"
      @visit-room="selectRoom"
    />
  </div>
  <div v-else>
    <ChatRoom
      :roomWebSocket="roomWebSocket"
      :roomMembers="roomMembers"
      :privacy="privateRoom"
      :userNotAllowedInRoom="userNotAllowedInRoom"
      :joinRequests="joinRequests"
      :leftRoom="leftRoom"
      :displayName="roomDisplayName"
      :authToken="authToken"
      :room="room"
      :userId="userId"
      @go-home="goHome"
    />
  </div>
</template>

<script>
import { v4 as uuidv4 } from "uuid";
import firebase from "firebase/app";
import PhoneSignIn from "./components/PhoneSignIn.vue";
import HomePage from "./components/HomePage.vue";
import ChatRoom from "./components/ChatRoom.vue";

export default {
  name: "App",
  components: {
    HomePage,
    ChatRoom,
    PhoneSignIn,
  },
  data() {
    return {
      authToken: "",
      userId: "",
      room: null,
      isAnonymous: true,
      verifyPhone: false,
      notifications: [],
      displayName: "",
      userWebSocket: null,
      roomWebSocket: null,
      roomMembers: [],
      privateRoom: false,
      userNotAllowedInRoom: "",
      joinRequests: [],
      leftRoom: "",
      roomDisplayName: "",
    };
  },
  methods: {
    verify: function () {
      this.verifyPhone = true;
    },
    cancelVerify: function () {
      this.verifyPhone = false;
    },
    signedIn: function (token, userId, isAnonymous) {
      this.authToken = token;
      this.userId = userId;
      this.isAnonymous = isAnonymous;
    },
    newRoom: function () {
      const room = uuidv4();
      const url = new URL(window.location.href.split("?")[0]);
      url.searchParams.set("room", room);
      window.history.replaceState("", "", url);
      const urlParams = new URLSearchParams(window.location.search);
      this.room = urlParams.get("room");
      this.roomDisplayName = "";
    },
    selectRoom: function () {
      const urlParams = new URLSearchParams(window.location.search);
      this.room = urlParams.get("room");
      this.roomDisplayName = "";
    },
    goHome: function () {
      const urlParams = new URLSearchParams(window.location.search);
      this.room = urlParams.get("room");
    },
    connectUserWebsocket: function () {
      const backendUrl = new URL(process.env.VUE_APP_BACKEND_URL);
      const ws_scheme = backendUrl.protocol == "https:" ? "wss" : "ws";
      const path =
        ws_scheme +
        "://" +
        backendUrl.hostname +
        ":" +
        backendUrl.port +
        "/ws/user/" +
        this.userId +
        "/?token=" +
        this.authToken;
      this.userWebSocket = new WebSocket(path);
      this.userWebSocket.onopen = () => {
        console.log("User WebSocket open");
      };
      this.userWebSocket.onmessage = (message) => {
        const data = JSON.parse(message.data);
        if ("notifications" in data) {
          this.notifications = data.notifications;
        } else if (data.type == "refresh_notifications") {
          this.userWebSocket.send(
            JSON.stringify({
              command: "fetch_notifications",
            })
          );
        } else if ("display_name" in data) {
          this.displayName = data.display_name;
          this.editableDisplayName = data.display_name;
        }
      };
      this.userWebSocket.onerror = (e) => {
        console.log(e.message);
      };
      this.userWebSocket.onclose = () => {
        console.log("User WebSocket closed");
        this.connectUserWebsocket();
      };
    },
    connectRoomWebSocket: function () {
      const backendUrl = new URL(process.env.VUE_APP_BACKEND_URL);
      const ws_scheme = backendUrl.protocol == "https:" ? "wss" : "ws";
      const path =
        ws_scheme +
        "://" +
        backendUrl.hostname +
        ":" +
        backendUrl.port +
        "/ws/room/?token=" +
        this.authToken;
      this.roomWebSocket = new WebSocket(path);
      this.roomWebSocket.onopen = () => {
        console.log("Room WebSocket open");
        if (this.room) {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "connect",
              room: this.room,
            })
          );
        }
      };
      this.roomWebSocket.onmessage = (message) => {
        const data = JSON.parse(message.data);
        if ("members" in data) {
          this.roomMembers = data.members;
        } else if (data.type == "refresh_members") {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "fetch_members",
            })
          );
        } else if ("privacy" in data) {
          this.privateRoom = data.privacy;
        } else if (data.type == "refresh_privacy") {
          this.roomWebSocket.send(JSON.stringify({ command: "fetch_privacy" }));
        } else if ("allowed" in data) {
          if (!data.allowed) {
            this.userNotAllowedInRoom = data.room;
          } else {
            this.userNotAllowedInRoom = "";
          }
        } else if (data.type == "refresh_allowed_status") {
          this.roomWebSocket.send(
            JSON.stringify({ command: "fetch_allowed_status" })
          );
        } else if ("join_requests" in data) {
          this.joinRequests = data.join_requests;
        } else if (data.type == "refresh_join_requests") {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "fetch_join_requests",
            })
          );
        } else if (data.type == "left_room") {
          this.leftRoom = data.room;
        } else if ("display_name" in data) {
          this.roomDisplayName = data.display_name;
        } else if (data.type == "refresh_display_name") {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "fetch_display_name",
            })
          );
        } else if (data.type == "room_notified") {
          this.roomWebSocket.send(
            JSON.stringify({
              command: "read_room_notification",
            })
          );
        }
      };
      this.roomWebSocket.onerror = (e) => {
        console.log(e.message);
      };
      this.roomWebSocket.onclose = () => {
        console.log("Room WebSocket closed");
        this.connectRoomWebSocket();
      };
    },
  },
  watch: {
    userId() {
      if (this.userWebSocket) {
        this.userWebSocket.close();
      } else {
        this.connectUserWebsocket();
      }
      if (this.roomWebSocket) {
        this.roomWebSocket.close();
      } else if (this.room) {
        this.connectRoomWebSocket();
      }
    },
    room(newRoom, oldRoom) {
      if (this.userId && newRoom && !oldRoom && this.roomWebSocket) {
        this.roomWebSocket.send(
          JSON.stringify({
            command: "connect",
            room: this.room,
          })
        );
      } else if (this.userId && newRoom && !oldRoom && !this.roomWebSocket) {
        this.connectRoomWebSocket();
      }
    },
  },
  mounted() {
    const urlParams = new URLSearchParams(window.location.search);
    this.room = urlParams.get("room");
    const firebaseConfig = {
      apiKey: process.env.VUE_APP_FIREBASE_API_KEY,
      authDomain: process.env.VUE_APP_FIREBASE_AUTH_DOMAIN,
      projectId: process.env.VUE_APP_FIREBASE_PROJECT_ID,
      storageBucket: process.env.VUE_APP_FIREBASE_STORAGE_BUCKET,
      messagingSenderId: process.env.VUE_APP_FIREBASE_MESSAGING_SENDER_ID,
      appId: process.env.VUE_APP_FIREBASE_APP_ID,
      measurementId: process.env.VUE_APP_FIREBASE_MEASUREMENT_ID,
    };
    firebase.initializeApp(firebaseConfig);
    firebase.auth().onAuthStateChanged((user) => {
      if (user) {
        user.getIdToken().then((token) => {
          this.token = token;
          this.authToken = token;
          this.userId = user.uid;
          this.isAnonymous = user.isAnonymous;
        });
      } else {
        firebase.auth().signInAnonymously();
      }
    });
  },
};
</script>

<style src="@/assets/toggle.css"></style>
<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
<style scoped>
.verify-button {
  padding: 6px 10px;
  border-radius: 50%;
  cursor: pointer;
}
.verify-button:hover {
  background: #e0e0e0;
}
</style>