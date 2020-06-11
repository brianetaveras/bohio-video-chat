<template>
  <div class="home">
    <video
      ref="friends-video"
      poster="https://techcrunch.com/wp-content/uploads/2016/10/tv-countdown.gif?w=730&crop=1"
      id="friends-video"
      autoplay
    ></video>
    <video
      ref="my-video"
      muted
      poster="https://techcrunch.com/wp-content/uploads/2016/10/tv-countdown.gif?w=730&crop=1"
      id="my-video"
      autoplay
    ></video>
    <div v-if="callPrompt" class="call-dialog">
      <h1>Someone is calling you</h1>
      <button @click="acceptCall">Accept Call</button>
    </div>
    <div class="online-users">
      <div @click="callFriend(user)" v-for="(user, index) in onlineUsers" :key="index" class="user">
        <span v-if="user == myId">(You)</span>
      </div>
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
import HelloWorld from "@/components/HelloWorld.vue";
import io from "socket.io-client";
import Peer from "simple-peer";
export default {
  name: "Home",
  data() {
    return {
      stream: null,
      io: null,
      myId: null,
      onlineUsers: {},
      callData: null,
      callPrompt: false
    };
  },
  components: {
    HelloWorld
  },
  created() {
    this.io = io("https://6ffb3096ee85.ngrok.io/");

    this.io.on("init", id => {
      this.myId = id;
    });

    this.io.on("usersOnline", users => (this.onlineUsers = users));

    this.io.on('callRequest', (data) =>{
      this.callPrompt = true;
      this.callData = {
        id: data.from,
        name: data.from,
        signal: data.signal
      }
    })
  },
  mounted() {
    navigator.mediaDevices
      .getUserMedia({ audio: true, video: true })
      .then(stream => {
        this.stream = stream;
        this.$refs["my-video"].srcObject = stream;
        this.$refs["my-video"].play()
      });
  },
  methods: {
    callFriend(id) {
      const peer = new Peer({
        initiator: true,
        trickle: false,
        config: {
          iceServers: [
            {
              urls: "stun:stun.l.google.com:19302"
            },
            {
              urls: "stun:global.stun.twilio.com:3478?transport=udp"
            }
          ],
          sdpSemantics: "unified-plan"
        },
        stream: this.stream
      });

      peer.on("signal", data => {
        this.io.emit("callUser", {
          userToCall: id, 
          signalData: data, 
          from: this.myId
        });
      });

      peer.on("stream", stream => {
        this.setFriendVideo(stream);
      });

      this.io.on("callAccepted", signal => {
        peer.signal(signal);
      });
    },
    acceptCall() {
      this.callPrompt = false;
      const peer = new Peer({
        initiator: false,
        trickle: false,
        stream: this.stream
      });

      peer.on("signal", data => {
        this.io.emit("acceptCall", { signal: data, to: this.callData.id });
      });

      peer.on("stream", stream => {
        this.setFriendVideo(stream);
      });

      peer.signal(this.callData.signal);
    },
    setFriendVideo(stream) {
      console.log(stream);
      this.$refs["friends-video"].srcObject = stream;
      this.$refs["friends-video"].play()
    }
  }
};
</script>


<style lang="scss">
* {
  margin: 0;
}

.call-dialog{
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  background: white;
  padding: 20px;
  border-radius: 15px;
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.2);

  button{
    width: 100%;
    margin: 10px 0;
    height: 50px;
  }
}

.home {
  position: relative;
}

video {
  margin: 0;
}

.online-users {
  position: absolute;
  top: 2%;
  left: 2%;

  .user {
    width: 50px;
    height: 50px;
    background: white;
    border-radius: 50%;
    align-content: center;
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 10px 0;
    background: url("https://picsum.photos/200/300");
    background-size: cover;
    background-position: center;
    cursor: pointer;

    span {
      color: red;
    }
  }
}

#my-video {
  position: absolute;
  top: 2%;
  right: 2%;
  height: 150px;
  width: 150px;
  object-position: center;
  object-fit: cover;
  box-sizing: 0 3px 6px;
  border-radius: 50%;
}

#friends-video {
  object-fit: cover;
  object-position: center;
  height: 100vh;
  width: 100vw;
}
</style>
