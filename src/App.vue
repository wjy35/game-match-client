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

      <button @click="enter">수락</button>
      <button @click="cancel">거절</button>
    </div>

    <div v-if="matchStatus===MatchStatus.ENTERED">
      <p>다른 참가자를 기다려 주세요!</p>
    </div>

    <div v-if="matchStatus===MatchStatus.STARTED">
      <p>게임이 시작되었습니다.</p>
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
      gameId:"",
      sessionId:"",
      matchStatus: MatchStatus.READY,
      game: {
        status:0,
      },
    }
  },
  methods: {
    match() {
      this.matchStatus = MatchStatus.MATCHING;
      console.log(process.env.VUE_APP_MATCH_ENDPOINT_URL);
      this.client = Stomp.client(process.env.VUE_APP_MATCH_ENDPOINT_URL);
      this.client.connect(
          {},
          ()=>{
            this.client.subscribe(
                "/user/queue/match",
                (frame)=>{
                  let response = JSON.parse(frame.body);
                  if(response.success){
                    this.matched = true;
                    this.matchStatus = MatchStatus.MATCHED
                    this.sessionId = response.sessionId;
                    this.gameId = response.gameId;
                  }
                },
                (error)=>{
                  console.log(error);
                });
            this.client.send("/start");
          },
          (error)=>{
            console.log(error)
          },
      )
    },
    cancel(){
      this.matchStatus = MatchStatus.READY;
      this.gameId = "";
      this.sessionId = "",
          this.client.disconnect();
    },
    enter(){
      this.matchStatus = MatchStatus.ENTERED;
      this.client.subscribe(`/topic/enter/${this.gameId}`,
          (frame)=>{
            let enterResult  = frame.body;
            if(enterResult){
              this.matchStatus = MatchStatus.STARTED;
              this.client.subscribe(`/topic/game/${this.gameId}`,
                  (frame)=>{
                    this.game.status = frame.body;
                  },
                  ()=>{}
              );
              this.client.send(`/load/${this.gameId}/${this.sessionId}`);
              this.client.unsubscribe(`/topic/enter/${this.gameId}`);
              this.client.unsubscribe("/user/queue/match");
            }
          },
          (error)=>{
            console.log(error);
          }
      );
      this.client.send(`/enter/${this.gameId}/${this.sessionId}`);
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
