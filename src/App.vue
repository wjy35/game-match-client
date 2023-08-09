<template>
  <div id="app">
    <div>
      <button @click="test">
        테스트 연결
      </button>
      <hr>
    </div>
    <div v-if="matchStatus===MatchStatus.READY">
      <button @click="match">매칭</button>
    </div>

    <div v-if="matchStatus===MatchStatus.MATCHING">
      <p>매칭중...</p>
      <button @click="cancel">취소</button>
    </div>

    <div v-if="matchStatus===MatchStatus.MATCHED">
      <p>게임에 참가 하시겠습니까?</p>
      <p>{{second}}</p>
      <button @click="enter">수락</button>
      <button @click="cancel">거절</button>
    </div>

    <div v-if="matchStatus===MatchStatus.CREATED">
      <p>잠시후 시작합니다.</p>
      <p>{{sessionId}}</p>
      <p>{{memberToken}}</p>
      <p>{{memberId}}</p>

      <button @click="start">시작</button>
      <hr>
      <div id="session" v-if="session">
        <div id="session-header">
          <h1 id="session-title">{{ memberId }}</h1>
        </div>

        <div id="main-video">
          <user-video v-if="subscribers.length>-1" :stream-manager="subscribers[0]" />
        </div>

        <div>
          <user-video  v-if="subscribers.length>0" :stream-manager="subscribers[1]"/>
        </div>



      </div>

      <div v-if="gameStatus===GameStatus.WAIT_GAME_START">
        <p>잠시후 게임 시작</p>
        <p>{{second}}</p>
      </div>

      <div v-if="gameStatus===GameStatus.PICK_TOPIC_TYPE">
        <p>대분류 뽑기</p>
        <p>{{second}}</p>
      </div>

      <div v-if="gameStatus===GameStatus.PICK_TOPIC_KEYWORD">
        <p>소분류 뽑기</p>
        <p>{{second}}</p>
      </div>
      <div v-if="gameStatus===GameStatus.PREPARE_PRESENT">
        발표 준비
        {{tellerToken}}
        {{second}}
        <button @click="skipPreparePresent">발표 준비 완료</button>
      </div>
      <div v-if="gameStatus===GameStatus.PRESENT">
        <p>발표중</p>
        {{second}}
      </div>
    </div>

  </div>
</template>

<script>

import {MatchStatus} from "@/match-status";
import {GameStatus} from "@/game-status";
import * as Stomp from "webstomp-client";
import UserVideo from "@/components/UserVideo.vue";
import { OpenVidu } from "openvidu-browser";

export default {
  name: 'App',
  components: {UserVideo},
  computed: {
    GameStatus() {
      return GameStatus
    },
    MatchStatus() {
      return MatchStatus
    }
  },
  data() {
    return {
      client : null,
      groupId:"",
      memberId:"",
      second: "",
      matchStatus: MatchStatus.READY,
      sessionId:"",
      memberToken:"",
      gameResponse:"",
      gameStatus:"",
      tellerToken:"",
      gameMemberOrder:"",
      topic:"",
      OV: undefined,
      session: undefined,
      mainStreamManager: undefined,
      publisher: undefined,
      subscribers: [],
    }
  },
  unmounted() {
    this.leaveSession();
  },
  methods: {
    updateMainVideoStreamManager(stream) {
      if (this.mainStreamManager === stream) return;
      this.mainStreamManager = stream;
    },
    match() {
      /**
       * Todo
       * 매칭 시작 버튼 비활성화 꼭 해주어야함!
       */

      this.matchStatus = MatchStatus.MATCHING;

      this.client = Stomp.client(process.env.VUE_APP_MATCH_ENDPOINT_URL);
      this.client.connect(
          {},
          ()=>{
            this.client.subscribe(
                "/user/queue/match",
                (frame)=>{
                  let response = JSON.parse(frame.body);
                  if(response.success){
                    this.matchStatus = response.matchStatus;

                    if(response.matchStatus===MatchStatus.MATCHED){
                      if(response.second){
                        this.second = response.second;
                      }else{
                        this.matchStatus = response.matchStatus;
                        this.memberId = response.memberId;
                        this.groupId = response.groupId;
                      }

                    }else if(response.matchStatus===MatchStatus.CREATED){
                      /**
                       * ToDo
                       * matchStatus 가 CREATED 상태가 되면 game page로 이동
                       * groupId, memberId 와 함께 넘어가야함
                       */
                      this.memberToken = response.memberToken;
                      this.sessionId = response.sessionId;

                    }
                  }
                },
                (error)=>{
                  /**
                   * ToDo
                   * 에러 처리
                   * 인터넷 연결, 브라우저 오류 등의 문제로 소켓 연결 실패시 실행되는 callback
                   */

                  console.log(error);
                });
          },
          (error)=>{
            /**
             * ToDo
             * 에러 처리
             * 인터넷 연결, 브라우저 오류 등의 문제로 소켓 연결 실패시 실행되는 callback
             */

            console.log(error)
          },
      )
    },
    cancel(){
      this.groupId = "";
      this.memberId = "";
      this.client.disconnect();
      this.matchStatus = MatchStatus.READY;
    },
    enter(){
      /**
       * ToDo
       * 수락 버튼을 누르면 수락 버튼을 꼭 비활성화 해주세요!
       */
      this.client.send(`/enter/${this.groupId}/${this.memberId}`);
    },
    start(){
      this.OV = new OpenVidu();
      this.session = this.OV.initSession();

      this.session.on("streamCreated", ({stream}) => {
        const subscriber = this.session.subscribe(stream);
        this.subscribers.push(subscriber);
        console.log("subscriber Token 시발아",this.getMemberTokenByStreamManager(subscriber));
        // this.subscribers.push({
        //   streamManager: subscriber,
        //   memberToken: ""
        // });
        // this.subscribers.push(subscriber);
      });

      this.session.on("streamDestroyed", ({stream}) => {
        const index = this.subscribers.indexOf(stream.streamManager, 0);
        if (index >= 0) {
          this.subscribers.splice(index, 1);
        }
      });

      this.session.on("exception", ({ exception }) => {
        console.warn(exception);
      });

      this.session.connect(this.memberToken, {
        clientData: this.memberToken
      }).then(() => {
            let publisher = this.OV.initPublisher(undefined, {
              audioSource: undefined, // The source of audio. If undefined default microphone
              videoSource: undefined, // The source of video. If undefined default webcam
              publishAudio: true, // Whether you want to start publishing with your audio unmuted or not
              publishVideo: true, // Whether you want to start publishing with your video enabled or not
              resolution: "640x480", // The resolution of your video
              frameRate: 30, // The frame rate of your video
              insertMode: "APPEND", // How the video is inserted in the target element 'video-container'
              mirror: false, // Whether to mirror your local video or not
            });

            this.publisher = publisher;
            this.subscribers.push(publisher);
        this.session.publish(this.publisher);

      })
          .catch((error) => {
            console.log("There was an error connecting to the session:", error.code, error.message);
          });

      this.client.subscribe(
          `/topic/game/${this.sessionId}`,
          (frame)=>{
            let gameResponse = JSON.parse(frame.body);
            // let gameResponseString = frame.body;
            this.gameStatus = gameResponse.gameStatus;
            this.second = gameResponse.second;

            if(gameResponse.gameMemberOrder){
              this.gameMemberOrder = gameResponse.gameMemberOrder;
              console.log(this.gameMemberOrder);
            }

            if(gameResponse.tellerToken){
              this.tellerToken = gameResponse.tellerToken;
            }

            if(gameResponse.topic){
              this.topic = gameResponse.topic;
            }
          },
          ()=>{

          },
      )
    },
    pick(){
      this.client.send(`/pick/shuffle/${this.sessionId}`,JSON.stringify(
          {
            memberToken:this.memberToken,
            type:1,
            keyword:"삼행시"
          }));
    },
    skipPreparePresent(){
      this.client.send(`/skip/prepare/${this.sessionId}`);
    },
    leaveSession() {
      if (this.session) this.session.disconnect();
      this.session = undefined;
      this.mainStreamManager = undefined;
      this.publisher = undefined;
      this.subscribers = [];
      this.OV = undefined;
    },
    getMemberTokenByStreamManager(streamManager){
      return JSON.parse(streamManager.stream.connection.data).clientData;
    },
  },
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
