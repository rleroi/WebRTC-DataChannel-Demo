<template>
  <div>
    <form @submit.prevent>
      <button @click="initiateConnection">Initiate connection</button> OR <button @click="waitForConnection">Wait for connection</button><br>

      <textarea v-model="iceCandidateJson" placeholder="please add each ice candidate separately" style="width: 500px;"></textarea>
      <button @click="addIceCandidate">add candidate</button><br>

      <textarea v-model="localDescriptionJson" placeholder="local description" style="width: 500px;"></textarea><br>

      <textarea v-model="remoteDescriptionJson" placeholder="remote description" style="width: 500px;"></textarea>
      <button @click="setRemoteDescription">set</button><br>

      <div v-if="dataChannel.readyState === 'open'">
        <textarea v-model="messageData" placeholder="Message"></textarea><button @click="sendData(messageData)">Send</button>
      </div>
    </form>
  </div>
</template>

<script>
  /*
  Google is kind enough to let people use this for demos, but remember that it is not suitable for public use and if you are building a real app for production use,
  you should look into setting up your own servers or using a commercial service like Xirsys to provide production ready STUN and TURN signaling for you.
   */
const config = {
  'iceServers': [
    {
      'url': 'stun:stun.l.google.com:19302'
    },
    {
      'url': 'turn:192.158.29.39:3478?transport=udp',
      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
      'username': '28224511:1379330808'
    },
    {
      'url': 'turn:192.158.29.39:3478?transport=tcp',
      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
      'username': '28224511:1379330808'
    }
  ]
};

const dataChannelOptions = {
  ordered: false, //no guaranteed delivery, unreliable but faster
};



export default {
  name: 'home',
  data() {
    return {
      localDescriptionJson: '',
      remoteDescriptionJson: '',
      iceCandidateJson: '',

      rtcPeerConnection: {},

      dataChannel: {},

      messageData: '',
    }
  },
  mounted() {

  },

  methods: {
    // initialize connection from this side
    initiateConnection() {
      this.rtcPeerConnection = new RTCPeerConnection(config);
      //this.rtcPeerConnection.ondatachannel = this.onDataChannel; // we init the connection, we dont need this, right?
      this.rtcPeerConnection.onicecandidate = this.onIceCandidate;

      // we create the dataChannel because we init the connection.
      this.dataChannel = this.rtcPeerConnection.createDataChannel('arbitraryLabel', dataChannelOptions);
      this.dataChannel.onopen = this.dataChannelOnOpen;

      // create an offer and set our local description
      this.rtcPeerConnection.onnegotiationneeded = this.createOffer;
    },
    // OR connection from the other side, use onDataChannel() to set this.dataChannel and use addIceCandidate() to add other side's candidates
    waitForConnection() {
      this.rtcPeerConnection = new RTCPeerConnection(config);
      this.rtcPeerConnection.ondatachannel = this.onDataChannel;
      //this.rtcPeerConnection.onicecandidate = this.onIceCandidate; // other side inits the connection, we dont need this, right?

      // we dont create a dataChannel because we didn't init the connection. we wait for the ondatachannel event to set it.

      // we wait for an offer to be set in remote description, then we can create an answer.
      console.log('waiting for remote description... (offer)');
    },

    addIceCandidate() {
      this.rtcPeerConnection.addIceCandidate(new RTCIceCandidate(JSON.parse(this.iceCandidateJson))).then(() => {
        console.log('Successfully added ICE candidate.');
      }).catch((e) => {
        console.log('Error adding ICE candidate.', e);
      });
    },

    onIceCandidate(e) {
      if(!e.candidate) {
        console.log('No candidate');
        return false;
      }

      console.log('ICE candidate, please send to remote:', JSON.stringify(e.candidate));
    },

    setRemoteDescription() {
      const sdp = new RTCSessionDescription(JSON.parse(this.remoteDescriptionJson));
      this.rtcPeerConnection.setRemoteDescription(sdp);

      // if the remote description is an 'offer', we need to create an answer
      if(this.rtcPeerConnection.remoteDescription.type === 'offer') {
        this.createAnswer();
      }
    },

    setLocalDescription(sdp) {
      this.rtcPeerConnection.setLocalDescription(sdp);

      this.localDescriptionJson = JSON.stringify(sdp);
    },

    createOffer() {
      this.rtcPeerConnection.createOffer().then(sdp => {
        // after creating our offer, set it as the localDescription. make sure the other peer uses it as remoteDescription (signaling)
        this.setLocalDescription(sdp);
        console.log('Offer created, please copy our localDescription and give it to the other peer to use as remoteDescription.');
      });
    },

    createAnswer() {
      this.rtcPeerConnection.createAnswer().then(sdp => {
        // after creating the answer, set our localDescription. make sure the other peer uses it as remoteDescription (signaling)
        this.setLocalDescription(sdp);
        console.log('Answer created, please copy our localDescription and give it to the other peer to use as remoteDescription.');
      });
    },





    // data channel
    onDataChannel(e) {
      this.dataChannel = e.channel;
      this.dataChannel.onopen = this.dataChannelOnOpen;
    },

    dataChannelOnOpen() {
      if(this.dataChannel.readyState === 'open') {
        console.log('DataChannel open!');
        this.dataChannel.onmessage = this.dataChannelOnMessage;
      } else {
        console.log('DataChannel not open:', this.dataChannel.readyState);
      }
    },

    dataChannelOnMessage(e) {
      console.log(e);
    },

    sendData(data) {
      this.dataChannel.send(data);
    }
  }
}
</script>
