<template>
  <div id="app">
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

      <div id="gameResponses"></div>
      <div>
        <button @click="pick">good</button>
      </div>
    </div>

  </div>
</template>

<script>

import {MatchStatus} from "@/match-status";
import * as Stomp from "webstomp-client";

export default {
  name: 'App',
  computed: {
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
    }
  },
  methods: {
    match() {
      /**
       * Todo
       * 매칭 시작 버튼 비활성화 꼭 해주어야함!
       */

      this.matchStatus = MatchStatus.MATCHING;

      // process.env.VUE_APP_MATCH_ENDPOINT_URI 는 /endpoint
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
      this.client.subscribe(
          `/topic/game/${this.sessionId}`,
          (frame)=>{
            // let gameResponse = JSON.parse(frame.body);
            let gameResponseString = frame.body;
            document.getElementById("gameResponses").innerHTML += `<p>${gameResponseString}</p>`;
          },
          ()=>{

          }
      )
    },
    pick(){
      this.client.send(`/pick/${this.sessionId}`,JSON.stringify({type:1,keyword:"삼행시"}));
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
