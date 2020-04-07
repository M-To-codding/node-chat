<template>
  <b-container>
    <b-row>
      <b-col cols="12">
        <h2>
          Chat Room
        </h2>

        <div class="pb-4">
          <b-button variant="outline-warning" @click="logout()">Logout</b-button>
        </div>

        <b-list-group class="panel-body" v-chat-scroll>
          <b-list-group-item v-for="(item, index) in chats" :key="index" class="chat">

            <!--Your message-->

            <div class="left clearfix" v-if="item.nickname === nickname">
              <b-img left src="http://placehold.it/50/55C1E7/fff&text=ME" rounded="circle" width="55" height="55" alt="img" class="m-1" />
              <div class="chat-body clearfix">
                <div class="header">
                  <strong class="primary-font">{{ item.nickname }}</strong> <small class="pull-right text-muted">
                  <span class="glyphicon glyphicon-time"/>{{ item.created_date | moment }}</small>
                </div>
                <p>{{ item.message }}</p>
              </div>
            </div>

            <!--Other message-->

            <div class="right clearfix" v-else>
              <b-img right src="http://placehold.it/50/55C1E7/fff&text=U" rounded="circle" width="55" height="55" alt="img" class="m-1" />
              <div class="chat-body clearfix">
                <div class="header">
                  <strong class="primary-font">{{ item.nickname }}</strong>
                  <small class="pull-right text-muted">
                    <span class="glyphicon glyphicon-time"/>
                      {{ item.created_date | moment }}
                  </small>
                </div>
                <p>{{ item.message }}</p>
              </div>
            </div>

          </b-list-group-item>

        </b-list-group>

        <small v-if="typing" class="typing">{{typing}} is typing</small>

        <ul v-if="errors && errors.length">
          <li v-for="(error, index) of errors" :key="index">
            {{error.message}}
          </li>
        </ul>

        <b-form @submit="onSubmit" class="chat-form mt-4">
          <b-input-group prepend="Message">
            <b-form-input id="message" v-model.trim="chat.message"/>
            <b-input-group-append>
              <b-btn type="submit" variant="outline-success">Send</b-btn>
            </b-input-group-append>
          </b-input-group>
        </b-form>

      </b-col>
    </b-row>
  </b-container>
</template>

<script>

import Vue from 'vue' // eslint-disable-line no-unused-vars
import * as io from 'socket.io-client'
import VueChatScroll from 'vue-chat-scroll' // eslint-disable-line no-unused-vars
import axios from 'axios'
import moment from 'moment'
Vue.use(VueChatScroll)

export default {
  name: 'ChatRoom',
  data () {
    return {
      chats: [],
      errors: [],
      nickname: this.$route.params.nickname,
      chat: {},
      typing: false,
      socket: io('http://localhost:5005')
    }
  },
  created () {
    axios.defaults.headers.common.Authorization = localStorage.getItem(
      'jwtToken'
    )
    axios.get('/api/chat/', this.$route.params.id)
      .then(response => {
        this.chats = response.data
      })
      .catch(e => {
        this.errors.push(e)
      })

    this.socket.on('typing', function (data) {
      this.typing = data
    }.bind(this))

    this.socket.on('job', function (data) {
      console.log(data)
    })
    console.log('data')
    this.socket.on('stopTyping', function () {
      console.log('stop')
      this.typing = false
    }.bind(this))

    this.socket.on('new-message', function (data) {
      if (data.message.room === this.$route.params.id) {
        this.chats.push(data.message)
      }
    }.bind(this))
  },
  watch: {
    'chat.message': function (value) {
      value ? this.socket.emit('typing', this.nickname) : this.socket.emit('stopTyping')
    }
  },
  methods: {
    logout (id) {
      this.socket.emit('save-message',
        {
          room: this.chat.room,
          nickname: this.chat.nickname,
          message: this.chat.nickname + ' left this room',
          created_date: moment
        })
      this.$router.push({
        name: 'JoinRoom',
        params: { id }
      })
    },
    onSubmit (event) {
      event.preventDefault()
      this.chat.room = this.$route.params.id
      this.chat.nickname = this.$route.params.nickname
      axios.post('/api/chat/', this.chat)
        .then(response => {
          this.socket.emit('save-message', response.data)
          this.chat.message = ''
        })
        .catch(e => {
          this.errors.push(e)
        })
    }
  },
  filters: {
    moment (value) {
      if (value) {
        return moment(String(value)).format('MM/DD/YYYY hh:mm')
      }
    }
  }

}
</script>

<style lang="scss">
  .chat .left .chat-body {
    text-align: left;
    margin-left: 100px;
  }

  .chat .right .chat-body {
    text-align: right;
    margin-right: 100px;
  }

  .chat .chat-body p {
    margin: 0;
    color: #777777;
  }

  .panel-body {
    overflow-y: scroll;
    height: 350px;
    box-shadow: 0 0 23px -12px rgba(0,0,0,0.75);
  }

  .chat-form {
    margin: 20px auto;
    width: 80%;
  }

  .typing {
    position: absolute;
    left: 0;
    right: 0;
     &:after{
       position: absolute;
       overflow: hidden;
       display: inline-block;
       vertical-align: bottom;
       animation: ellipsis steps(4,end) 900ms infinite;
       content: "\2026";
       width: 0;
     }
  }

  @keyframes ellipsis {
    to {
      width: 20px;
    }
  }
  @-webkit-keyframes ellipsis {
    to {
      width: 20px;
    }
  }
</style>