<template>
    <div class="w-screen h-screen flex flex-col">
        {{socket_id}}
        {{users}}
        <video ref="localVideo" class="vid" autoplay muted></video>
        <button id="switchButton" class="settings" @click="switchMedia()">Switch Camera</button>
        <button id="muteButton" class="settings" @click="toggleMute()">Unmuted</button>
        <button id="vidButton" class="settings" @click="toggleVid()">Video Enabled</button>
        <div class="grid grid-cols-4" ref="videos">
            
        </div>
    </div>
</template>

<script>
import SimplePeer from 'simple-peer';

export default {
    data() {
        return {
            socket_id: '',
            users: [],
            socket: {},
            localStream: null,
            peers: {},
            configuration: {
                // Using From https://www.metered.ca/tools/openrelay/
                "iceServers": [
                    {
                        urls: "stun:openrelay.metered.ca:80"
                    },
                    {
                        urls: "turn:openrelay.metered.ca:80",
                        username: "openrelayproject",
                        credential: "openrelayproject"
                    },
                    {
                        urls: "turn:openrelay.metered.ca:443",
                        username: "openrelayproject",
                        credential: "openrelayproject"
                    },
                    {
                        urls: "turn:openrelay.metered.ca:443?transport=tcp",
                        username: "openrelayproject",
                        credential: "openrelayproject"
                    }
                ]
            },
            constraints: {
                audio: true,
                video: {
                    width: {
                        max: 300
                    },
                    height: {
                        max: 300
                    }
                }
            },

        }
    },
    mounted() {
        /** SWITCH TO FRONT CAMERA */
        this.constraints.video.facingMode = {
            ideal: "user"
        }

        this.startCamera();
    },
    methods: {
        startCamera() {
            // enabling the camera at startup
            navigator.mediaDevices.getUserMedia(this.constraints).then(stream => {
                console.log('Received local stream');

                this.$refs.localVideo.srcObject = stream;
                this.localStream = stream;

                this.init()

            }).catch((e) => {
                console.error(e);
                // alert(`getusermedia error ${e.name}`)
                // this.init();
            })
        },
        init() {
            console.log('INIT');
            this.socket = this.$nuxtSocket({
                name: 'main',
            })
            this.socket_id = 'test';

            this.socket.on('initReceive', socket_id => {
                console.log('INIT RECEIVE ' + socket_id)
                this.addPeer(socket_id, false)

                this.socket.emit('initSend', socket_id)
            })

            this.socket.on('initSend', socket_id => {
                console.log('INIT SEND ' + socket_id)
                this.addPeer(socket_id, true)
            })

            this.socket.on('removePeer', socket_id => {
                console.log('removing peer ' + socket_id)
                this.removePeer(socket_id)
            })

            this.socket.on('disconnect', () => {
                console.log('GOT DISCONNECTED')
                for (let socket_id in peers) {
                    this.removePeer(socket_id)
                }
            })

            this.socket.on('signal', data => {
                console.log('SIGNAL RECEIVED')
                this.peers[data.socket_id].signal(data.signal)
            })
        },
        addPeer(socket_id, am_initiator) {
            console.log('ADD PEER');
            this.users[socket_id] = socket_id;
            this.peers[socket_id] = new SimplePeer({
                initiator: am_initiator,
                stream: this.localStream,
                config: this.configuration
            })

            this.peers[socket_id].on('signal', data => {
                this.socket.emit('signal', {
                    signal: data,
                    socket_id: socket_id
                })
            })

            this.peers[socket_id].on('stream', stream => {
                let newVid = document.createElement('video')
                newVid.srcObject = stream
                newVid.id = socket_id
                newVid.playsinline = false
                newVid.autoplay = true
                newVid.className = "vid"
                newVid.onclick = () => openPictureMode(newVid)
                newVid.ontouchstart = (e) => openPictureMode(newVid)
                this.$refs.videos.appendChild(newVid)
            })
        },
        removePeer(socket_id) {
            console.log('REMOVE PEER');

            let videoEl = document.getElementById(socket_id)
            if (videoEl) {
                const tracks = videoEl.srcObject.getTracks();

                tracks.forEach(function (track) {
                    track.stop()
                })

                videoEl.srcObject = null
                videoEl.parentNode.removeChild(videoEl)
            }
            if (this.peers[socket_id]) this.peers[socket_id].destroy()
            delete this.peers[socket_id]
            delete this.users[socket_id];
        },
        openPictureMode(el) {
            console.log('opening pip')
            el.requestPictureInPicture()
        },
        switchMedia() {
            if (this.constraints.video.facingMode.ideal === 'user') {
                this.constraints.video.facingMode.ideal = 'environment'
            } else {
                this.constraints.video.facingMode.ideal = 'user'
            }

            const tracks = this.localStream.getTracks();

            tracks.forEach(function (track) {
                track.stop()
            })

            this.$refs.localVideo.srcObject = null
            navigator.mediaDevices.getUserMedia(this.constraints).then(stream => {

                for (let socket_id in this.peers) {
                    for (let index in this.peers[socket_id].streams[0].getTracks()) {
                        for (let index2 in stream.getTracks()) {
                            if (this.peers[socket_id].streams[0].getTracks()[index].kind === stream.getTracks()[index2].kind) {
                                this.peers[socket_id].replaceTrack(this.peers[socket_id].streams[0].getTracks()[index], stream.getTracks()[index2], this.peers[socket_id].streams[0])
                                break;
                            }
                        }
                    }
                }

                this.localStream = stream
                this.$refs.localVideo.srcObject = stream
                // updateButtons()
            })
        },
        setScreen() {
            navigator.mediaDevices.getDisplayMedia().then(stream => {
                for (let socket_id in this.peers) {
                    for (let index in this.peers[socket_id].streams[0].getTracks()) {
                        for (let index2 in stream.getTracks()) {
                            if (this.peers[socket_id].streams[0].getTracks()[index].kind === stream.getTracks()[index2].kind) {
                                this.peers[socket_id].replaceTrack(this.peers[socket_id].streams[0].getTracks()[index], stream.getTracks()[index2], this.peers[socket_id].streams[0])
                                break;
                            }
                        }
                    }

                }
                this.localStream = stream
                this.$refs.localVideo.srcObject = stream;
                this.socket.emit('removeUpdatePeer', '')
            })
        },
        removeLocalStream() {
            if (this.localStream) {
                const tracks = this.localStream.getTracks();

                tracks.forEach(function (track) {
                    track.stop()
                })

                this.$refs.localVideo.srcObject = null
            }

            for (let socket_id in peers) {
                this.removePeer(socket_id)
            }
        },
        toggleMute() {
            for (let index in this.localStream.getAudioTracks()) {
                this.localStream.getAudioTracks()[index].enabled = !this.localStream.getAudioTracks()[index].enabled
                // muteButton.innerText = this.localStream.getAudioTracks()[index].enabled ? "Unmuted" : "Muted"
            }
        },
        toggleVid() {
            for (let index in this.localStream.getVideoTracks()) {
                this.localStream.getVideoTracks()[index].enabled = !this.localStream.getVideoTracks()[index].enabled
                // vidButton.innerText = this.localStream.getVideoTracks()[index].enabled ? "Video Enabled" : "Video Disabled"
            }
        }
    }
}
</script>

<style>
.vid {
    flex: 0 1 auto;
    height: 400px;
}

.settings {
    background-color: #4CAF50;
    border: none;
    color: white;
    padding: 5px 10px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 14px;
    margin: 2px 2px;
    cursor: pointer;
}
</style>