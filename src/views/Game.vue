<template>
  <div>
    <div class="header">
        <div class="columns is-mobile">
            <div class="column">
                Team: {{name_team}}
            </div>
            <div class="column has-text-centered">
                <span :class="statusServer">{{statusServer}}</span>
            </div>
            <div class="column has-text-right">
                Hint: {{hint.length}}
            </div>
        </div>
    </div>
    <div class="container">
        <div :class="classTimer">
            {{remainTime}}
        </div>
        <div class="progress-box">
            <progress :class="progress_bar.class" :value="progress_bar.value" max="100"></progress>
        </div>
        <div class="columns">
            <div class="column is-full">
                <div class="has-text-centered message-box">
                    {{message}}
                </div>
            </div>
        </div>
    </div>
    <div class="menu">
        <div class="columns is-mobile">
            <div class="outer column">
                <button class="hint" @click="callHint" :disabled="!statusGame">
                    <font-awesome-icon icon="lightbulb" size="2x" />
                    <div class="title-btn">Hint</div></button>
            </div>
            <div class="outer column ">
            <button class="penalty" @click="callPenalty" :disabled="!statusGame">
                <font-awesome-icon icon="skull-crossbones" size="2x" />
                <div class="title-btn">Penalty</div>
                </button>
                <!-- !statusGame &&  -->
            </div>
            <div class="outer column">
                <button class="code" @click="callCode" :disabled="!statusGame || checkFlagPenalty">
                    <font-awesome-icon icon="lock" size="2x" />
                    <div class="title-btn">Code {{countTimePenalty}}</div></button>
            </div>
        </div>
        <div class="columns is-mobile">
            <div class="outer column ">
                <button class="review" @click="reviewHint"  :disabled="!statusGame">
                    <font-awesome-icon icon="search" size="2x" /><br>
                    <div class="title-btn">Review Hints</div></button>
            </div>
            <div class="outer column ">
                <button class="help-staff" @click="callHelp"  :disabled="!statusGame">
                    <font-awesome-icon icon="question" size="2x" /><br>
                    <div class="title-btn">Help</div></button>
            </div>
            <div class="outer column ">
                <button class="machine" @click="callMachine" :disabled="!statusGame">
                    <font-awesome-icon icon="cog" size="2x"  /><br>
                    <div class="title-btn">Machine</div></button>
            </div>
        </div>
    </div>
    
    <div v-bind:class="{modal: true, 'is-active': show_hint }">
        <div class="modal-background"></div>
        <div class="modal-content" style="background: #fff;padding: 10px;text-align: center;">
            <div style="font-size:1.5em;color:#888;padding: 10px;">เลือกหมายเลขที่ต้องการ</div>
            <button class="button" v-for="hint_item in hint" v-on:click="callHintNum(hint_item)" >
                {{ hint_item }}
            </button>
        </div>
        <button @click="closeBtn" class="modal-close is-large" ></button>
    </div>
    <div v-bind:class="{modal: true, 'is-active': game_puzzle }">
        <div class="modal-background"></div>
        <div class="modal-content">
            <puzzle></puzzle>
        </div>
        <button  @click="closeBtn" class="modal-close is-large" ></button>
    </div>
    <div v-bind:class="{modal: true, 'is-active': game_musicbox }">
        <div class="modal-background"></div>
        <div class="modal-content">
            <music-box></music-box>
        </div>
        <button  @click="closeBtn" class="modal-close is-large" ></button>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client'
import axios from 'axios'
import userRoom from '../../db/user.json'

// import keyboard from 'vue-keyboard'
import puzzle from './Puzzle'
import musicBox from './Musicbox'
const HTTP_HOST = process.env.VUE_APP_HTTP_HOST

export default {
    //name: 'App',
    components: { puzzle, musicBox },    
    data() {
        return {
            classTimer: ['timer'],
            statusServer: 'wait..',
            statusGame: false,
            timeStart: null,
            timeFinish: null,
            remainTime: 'wait',
            team: '',
            message: 'รอสัญญาณเริ่มเกม ... ขอให้โชคดี :)',
            socket: io(HTTP_HOST),
            interval: null,
            game_puzzle: false,
            game_musicbox: false,
            show_hint: false,
            numPenalty: 0,
            timePenalty: 0,
            countTimePenalty: '',
            checkFlagPenalty: false,
            name_team: '',
            hint: [],
            progress_bar: {
                value: null,
                class: ['progress', 'is-small', 'is-theme']
            }
        }
    },
    mounted() {
        if(localStorage.getItem('timePenalty')){
            this.timePenalty = localStorage.getItem('timePenalty')
        }
        if(!localStorage.getItem('login')) {
            this.$router.push('/') 
        }
        this.team = localStorage.getItem('login')
        this.name_team = userRoom[this.team].team
        
        this.socket.on('getSettingGame', (data) => {
            this.statusGame = data.setting_game.gameStart
            this.timeFinish = data.setting_game.timeFinish
            this.hint = (data.hint != undefined) ? data.hint : []
            if(this.statusGame == true) {
                this.message = 'เริ่มเกมได้'
            }
        });
        this.socket.on('getStatusGame', (data) => {
            this.statusGame = data.gameStart
            this.timeStart = data.timeStart
            this.timeFinish = data.timeFinish
        });
        this.socket.on('connect', () => {
            if(this.socket.connected){
                this.statusServer = "online"
                this.socket.emit('LOGIN', {
                    team: this.team,
                    clientId: this.socket.id
                });
            }
            this.callInfoGame()
        })
        this.socket.on('disconnect', (reason) => {
            this.statusServer = "offline"
            console.log(reason)
            if (reason === 'io server disconnect') {
                // the disconnection was initiated by the server, you need to reconnect manually
                this.socket.connect();
            }
            // else the socket will automatically try to reconnect
        });
        this.socket.on('GAME', (data) => {
            if(data == 'pause') {
                this.message = 'หยุดเวลาชั่วคราว'
                clearInterval(this.interval)
            } else if(data == 'resume'){
                this.message = 'เริ่มเกมได้'
                this.countdownTime()
            } else if(data == 'reset'){
                clearInterval(this.interval)
                this.remainTime = '60:00'
            } else if(data == 'start'){
                this.message = 'เริ่มเกมได้'
                this.countdownTime()
            }
        });
        this.socket.on('updateHint', (data) => {
            this.hint = data
        })
        this.socket.on('MSG', (data) => {
            this.$swal({
                html:'<span style="color:#aaa;">ข้อความจากพระเจ้า</span><br><span style="font-size:2em;">'+data+'</span>'
            })
        })
        this.socket.on('STATUS', (data) => {
            this.message = data
        })
        
        this.countdownTime()
    },
    methods: {
        callHelp() {
            axios.get(`${HTTP_HOST}/help/${this.team}`)
            .then(result => {
                this.$swal({
                    title: 'ขณะนี้ได้เรียกทีมงานแล้ว',
                    text: 'รอแปปนึงนะ :)'
                })
            })
        },
        callPenalty() {
            axios.get(HTTP_HOST+'/penalty/'+this.team)
                .then(response => {
                    this.todos = response.data
                })
                .catch(error => {
                console.log(error);
            }).then(() => {
                // this.message = 'คุณถูกระงับไม่ให้ใส่รหัส 1 นาที'
                this.timePenalty = Math.floor(Date.now() / 1000) + 60
                localStorage.setItem("timePenalty", this.timePenalty);
                this.socket.emit('PENALTY', this.team)
                this.numPenalty = 0
                this.$swal({
                    text: 'คุณถูกลงโทษเพิ่ม 1 ครั้ง'
                })
            })
        },
        checkPenalty() {
            if(this.timePenalty > Math.floor(Date.now() / 1000)) {
                this.countTimePenalty = "("+(this.timePenalty-Math.floor(Date.now() / 1000))+")"
                this.checkFlagPenalty = true
                return true
            }
            this.countTimePenalty = ''
            // this.message = 'เริ่มเกมได้'
            this.checkFlagPenalty = false
            return false
        },
        countdownTime() {
            this.interval = setInterval(() => {
                if(this.statusServer == "online" && this.statusGame == true) {
                    this.calculateTime()
                } else {
                    this.remainTime = '00:00'
                }
            },1000);
        },
        callInfoGame() {
            this.socket.emit('SETTING', this.team);
        },
        calculateTime() {
            function pad(d) { return (d < 10) ? '0' + d.toString() : d.toString(); }
            // var date = Math.floor(Date.now() / 1000)
            this.timeStart = Math.floor(Date.now() / 1000)
            var time = Math.abs(parseInt(this.timeFinish)-parseInt(this.timeStart))
            this.progress_bar.value = Math.floor(100*time/3600)
            var minutes = Math.floor(time / 60)
            var seconds = time - minutes * 60
            this.remainTime = pad(minutes)+":"+pad(seconds)
            this.checkPenalty()
            if(minutes == 0 && seconds == 0) {
                this.$swal({
                    title: 'หมดเวลาแล้ว ภารกิจล้มเหลว'
                })
            }
            if( (minutes > 10) && (this.timeFinish > this.timeStart) ){
                this.classTimer = ['timer']
                this.progress_bar.class = ['progress', 'is-small', 'is-theme']
            } else {
                this.classTimer = ['timer', 'is-danger']
                this.progress_bar.class = ['progress', 'is-small', 'is-danger']
            }
            
        },
        sendMessage(e) {
            e.preventDefault();
            
            this.socket.emit('SEND_MESSAGE', {
                // user: this.user,
                message: this.message
            });
            this.message = ''
        },
        reviewHint() {
            this.show_hint = true
        },
        callHintNum(id) {
            axios.get(`${HTTP_HOST}/hint/${id}/${this.team}`)
            .then(result => {
                this.show_hint = false
                this.$swal({
                    title: 'คำใบ้หมายเลข '+id,
                    text: result.data.hint
                })
            })
        },
        callHint() {
            this.$swal({
                title: 'Hint',
                text: 'ใส่หมายเลขการ์ดที่ต้องการขอคำใบ้',
                input: 'number',
                inputAttributes: {
                    autocapitalize: 'off'
                },
                cancelButtonText: 'ยกเลิก',
                showCancelButton: true,
                confirmButtonText: 'ตกลง',
                showLoaderOnConfirm: true,
                preConfirm: (id) => {
                    return fetch(`${HTTP_HOST}/hint/${id}/${this.team}`)
                    .then(response => {
                        if (!response.ok) {
                            this.$swal({
                                type: 'error',
                                title: 'ไม่พบหมายเลขการ์ดนี้'})
                        }
                        return response.json()
                    })
                },
                allowOutsideClick: () => !this.$swal.isLoading()
                }).then((result) => {
                    if (result.value.hint) {
                        this.$swal({
                            title: 'คำใบ้หมายเลข '+result.value.id,
                            text: result.value.hint
                        })
                    }
                })
        },
        callMachine() {
            this.$swal({
                title: 'ใส่หมายเลขการ์ด',
                input: 'number',
                inputAttributes: {
                    autocapitalize: 'off'
                },
                cancelButtonText: 'ยกเลิก',
                showCancelButton: true,
                confirmButtonText: 'ตกลง',
                allowOutsideClick: () => !this.$swal.isLoading()
                }).then((code) => {
                    if(code.value == 40){ //musicbox
                        this.game_musicbox = true
                    } else if(code.value == 31){ //puzzle
                        this.game_puzzle = true
                    } else {
                        if(!code.dismiss) {
                            this.$swal({
                            type: 'error',
                            title: 'ไม่พบหมายเลขการ์ดนี้'})
                        }
                    }
                })
        },
        callCode() {
            var parent = this
            this.$swal({
                title: 'Code',
                text: 'กรุณาใส่รหัสผ่าน 4 ตัว',
                input: 'number',
                inputAttributes: {
                    autocapitalize: 'off'
                },
                cancelButtonText: 'ยกเลิก',
                showCancelButton: true,
                confirmButtonText: 'ตกลง',
                showLoaderOnConfirm: true,
                preConfirm: (id) => {
                    return fetch(`${HTTP_HOST}/code/${id}`)
                    .then(response => {
                        if (!response.ok) {
                            this.$swal({
                                type: 'error',
                                title: 'รหัสผ่านไม่ถูกต้อง'})
                            this.numPenalty++
                            if(this.numPenalty == 3){
                                this.message = 'คุณถูกระงับไม่ให้ใส่รหัส 2 นาที'
                                this.timePenalty = Math.floor(Date.now() / 1000) + 120
                                localStorage.setItem("timePenalty", this.timePenalty);
                                this.socket.emit('PENALTY', this.team)
                                this.numPenalty = 0
                            }
                            throw new Error("testset")
                        }
                        return response.json()
                    })
                    .catch()
                },
                allowOutsideClick: () => !this.$swal.isLoading()
                }).then((result) => {
                    if(result.value.id == '5130') {
                        this.socket.emit('COMPLETE', this.team)
                        clearInterval(parent.interval)
                        this.message = 'หนีออกได้สำเร็จ'
                    }
                    if (result.value.result) {
                        this.$swal({
                            title: result.value.result,
                        })
                    }
                })
        },
        closeBtn() {
            this.game_puzzle = false
            this.game_musicbox = false
            this.show_hint = false
        }
    },

}
</script>