<template>
  <div class="home">
    <button @click="startServer">Start server (Listener/Receiver)</button><br>
    <button @click="startClient">Start client (Sender)</button><br>
    status: {{status}}<br>
    data:<br>
    <textarea v-model="data" cols="100" rows="10"></textarea><br>
    <button v-if="!isServer" @click="sendDataToServer">Send data to server</button>

    <hr>
    <textarea v-model="descriptionData" cols="100" rows="10" placeholder="Description data, needed to establish a peer to peer connection"></textarea><br>
    <button @click="isServer ? addClientDescription() : addServerDescription()">Add description</button>
  </div>
</template>

<script>
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

export default {
  name: 'home',
  data() {
    return {
      server: null,
      client: null,
      status: '',
      data: '',
      isServer: true,
      sendChannel: null,
      receiveChannel: null,
      descriptionData: ''
    }
  },
  mounted() {

  },
  methods: {
    startServer() {
      this.isServer = true;

      // init server

      this.server = new RTCPeerConnection(config);

      this.server.onconnectionstatechange = (e) => {
        this.status = 'server connectionState: '+this.server.connectionState;
        console.log('connectionstatechange on server. state: '+this.server.connectionState);
      };

      // add ice servers
      //this.initIce(this.server);

      // add ICE candidates
      console.log('setting onicecandidate');
      this.server.onicecandidate = (e) => {
        if(!e.candidate) {
          console.log('No candidate');
          return false;
        }
        this.status = 'ICE candidate: '+e.candidate;
        console.log('ICE candidate: ', e.candidate);
        this.server.addIceCandidate(e.candidate).then(() => {
          this.status = "Successfully added ICE candidate.";
          console.log("Successfully added ICE candidate");
        }).catch(e => {
          this.status = 'Error adding ICE candidate.';
          console.log('Error adding ICE candidate');
        });
      };

      // server events
      this.server.ondatachannel = (e) => {
        console.log('Server received datachannel!', e);
        this.status = 'Server received dataChannel!';

        this.receiveChannel = e.channel;

        this.receiveChannel.onopen = (e) => {
          console.log('Listening on datachannel!', e);
          this.status = 'Listening on datachannel!'
        };

        this.receiveChannel.onmessage = e => {
          console.log('Received message from client!', e);
          this.status = 'Received message from client!';
          this.data = e.data;
        };
      };

      // create Offer
      this.server.createOffer().then(description => {
        console.log('server created an Offer, please send the description to the server', description);
        this.descriptionData = "Created description, server needs this to allow communication from us: "+JSON.stringify(description);

        this.server.setLocalDescription(description).then(() => {
          console.log('Server successfully set local description');
        }).catch(e => {
          console.log('Error setting server\'s local description', e);
        });
      });

    },

    startClient() {
      this.isServer = false;

      // init client

      this.client = new RTCPeerConnection(config);

      this.client.onconnectionstatechange = (e) => {
        this.status = 'client connectionState: '+this.client.connectionState;
        console.log('connectionstatechange on client. state: '+this.client.connectionState);
      };

      // create a data channel for sending data
      this.sendChannel = this.client.createDataChannel('sendDataChannel');

      // add ice servers
      //this.initIce(this.client);

      // add ICE candidates
      console.log('setting onicecandidate');
      this.client.onicecandidate = (e) => {
        if(!e.candidate) {
          console.log('No candidate');
          return false;
        }
        this.status = 'ICE candidate: '+e.candidate;
        console.log('ICE candidate: ', e.candidate);
        this.client.addIceCandidate(e.candidate).then(() => {
          this.status = "Successfully added ICE candidate.";
          console.log("Successfully added ICE candidate");
        }).catch(e => {
          this.status = 'Error adding ICE candidate.';
          console.log('Error adding ICE candidate', e);
        });
      };

      // when client is ready to send data
      this.sendChannel.onopen = () => {
        this.status =  'client.onopen, readyState: '+this.client.readyState;
        console.log('client.onopen, readyState: '+this.client.readyState);
      };
      this.sendChannel.onclose = () => {
        this.status =  'client.onclose, readyState: '+this.client.readyState;
        console.log('client.onclose, readyState: '+this.client.readyState);
      };

      // start

      // create Offer
      this.client.createOffer().then(description => {
        console.log('Client created an Offer, please send the description to the server', description);
        this.descriptionData = "Created description, server needs this to allow communication from us: "+JSON.stringify(description);

        this.client.setLocalDescription(description).then(() => {
          console.log('client successfully set local description');
        }).catch(e => {
          console.log('Error setting client\'s local description', e);
        });
      });
    },
    /*initIce(peer) {
      // add ICE candidates
      console.log('setting onicecandidate');
      peer.onicecandidate = (e) => {
          this.status = 'ICE candidate: '+e.candidate;
          console.log('ICE candidate: '+e.candidate);
          peer.addIceCandidate(e.candidate).then(() => {
            this.status = "Successfully added ICE candidate.";
            console.log("Successfully added ICE candidate");
          }).catch(e => {
            this.status = 'Error adding ICE candidate.';
            console.log('Error adding ICE candidate');
          });
      };
    },*/
    sendDataToServer() {
      this.sendChannel.send(this.data);
    },
    addClientDescription() {
      console.log(this.descriptionData);
      this.server.setRemoteDescription(JSON.parse(this.descriptionData)).then(() => {
        console.log('server set remote description AKA added client description to server');
      }).catch(e => {
        console.log('Error setting client description on server', e.message);
      });

      // creates an answer to the client's Offer (the info is in the description, description contains the SDP)
      this.server.createAnswer().then(description => {
        this.descriptionData = "Created description, client needs this to communicate with us: "+JSON.stringify(description);

        this.server.setLocalDescription(description).then(() => {
          console.log('server set local description from our Answer to the client\'s Offer', description);
        });
      });
    },
    addServerDescription() {
      console.log(this.descriptionData);
      this.client.setRemoteDescription(JSON.parse(this.descriptionData)).then(() => {
        console.log('Client set remote description from server. should be done, but did we see a connectionstatechange or readystatechange?');
      }).catch(e => {
        console.log('Error adding server description on client.', e.message);
      });
    }
  }
}
</script>
