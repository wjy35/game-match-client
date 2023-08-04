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
    }
  },
  methods: {
    match() {
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
                    if(response.second){
                      this.second = response.second;
                    }else{
                      this.matchStatus = response.matchStatus;
                      this.memberId = response.memberId;
                      this.groupId = response.groupId;

                      // matchStatus 가 created 상태라면 game page로 memberId + groupId 와 함께 넘어가야함
                    }
                  }
                },
                (error)=>{
                  console.log(error);
                });
          },
          (error)=>{
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
      this.client.send(`/enter/${this.groupId}/${this.memberId}`);
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
