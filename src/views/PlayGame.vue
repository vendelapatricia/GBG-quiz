<template>
  <div id ="content">
    <h1>{{theQuestion}}</h1>
    <input :maxlength="4" placeholder="Try your luck!" id="guess" class="field" @input="newValue"
    oninput="javascript: if (this.value.length > this.maxLength) this.value = this.value.slice(0, this.maxLength);" type="number"  ref ="focused" autofocus="this.value=''" v-on:keypress.enter = "makeGuess"  v-on:keypress = "OnlyNumbers"/>
    <button class="guessbutton" @click="makeGuess">Make a guess</button>
    <p id="errormess" style="color: orangered; display: none"><br>Only numbers! </p>
    <p id="playerTurn" v-show="playersTurn">Your turn! </p>
    <p id="botTurn" v-show="!playersTurn">Bot's turn! </p>
    <p id="guessHigher" v-visible="guessHigher">Guess higher!</p>
    <p id="guessLower" v-visible="guessLower">Guess lower! </p>

    <Timer v-show="show" ref="form"/>
    <div class="botText">
      <p class="specifikBot">My guess: <br>{{value}}</p>
      <p class="specifikBot" id="bot"> {{choosenBot}} guess: <br>{{bot}} </p>
    </div>
    <br>
  </div>
</template>


<script>
import Timer from '@/components/Timer.vue';
import {db, fb} from '../firebase-config';
import Vue from 'vue';
import VueFlashMessage from 'vue-flash-message';
import { timeout } from 'q';
import { functions } from 'firebase';
import VueVisible from 'vue-visible';
Vue.use(VueVisible);
Vue.use(VueFlashMessage);


export default {
  name: "PlayGame",
  data: function() {
    return {
      numberKeys: [],
      show: true,
      botGuesses: [],
      highLow: '',
      userGuess: 0,
      playersTurn: true,
      guessHigher: false,
      guessLower: false
    }
  },
  firebase: {
  questions: db.ref('questions'),
  allUsers: db.ref('allUsers').orderByChild("newPoint")
  },
  components: {
    Timer,
  },
  computed: {
    value() {
      return this.$store.getters.value;
    },
    number() {
      return this.$store.state.number;
    },
    bot() {
      return this.$store.state.bot;
    },
    uid() {
    return fb.auth().currentUser.uid;
    },
    user() {
      return fb.auth().currentUser;
    },
    oldScore() {
    return this.allUsers[this.uid].newPoint;
    },
    theQuestion () {
      return this.$store.state.theQuestion;
    },
    theAnswer() {
      return this.$store.state.theAnswer;
    },
    num() {
      return  this.$store.state.num;
    },
    arr() {
     return this.$store.state.arr;
    },
    choosenBot() {
      return this.$store.state.choosenBot;
    },
    maxGuess() {
      return this.$store.state.maxGuess;
    },
    minGuess() {
      return this.$store.state.minGuess;
    },
     sessionScore() {
       return this.$store.state.sessionScore;
    }
  },
  created () {
    this.$bindAsObject('allUsers', db.ref('allUsers/'));
  },
  methods: {
    setFocus()
    {
      this.$refs.focused.focus();
    },
    storeData() {
      this.$firebaseRefs.allUsers.child(this.uid).update({
      newPoint: parseInt(this.oldScore) + parseInt(10)});
    },
    stop() {
      this.$refs.form.stop()
    },
    start() {
      this.$refs.form.start()
    },
    reset() {
      this.$refs.form.reset()
    },
    newValue(event) {
      this.$store.dispatch('newValue', event.target.value)
    },
    decideMinMax() {
      if (this.$store.state.choosenBot == "Glenn") {
        this.$store.commit('updateMin', 6);
        this.$store.commit('updateMax', 30);
      } else if (this.$store.state.choosenBot == "Håkan") {
        this.$store.commit('updateMin', 6);
        this.$store.commit('updateMax', 15);
      } else {
        this.$store.commit('updateMin', 2);
        this.$store.commit('updateMax', 6);
    }
    },
    calculateScore() {
        if (this.$store.state.choosenBot == "Glenn" && this.$store.state.winner == true) {
          this.$store.commit('updateSessionScore', Math.floor((this.$store.state.numOfGuesses * 2) + 10));
        } else if (this.$store.state.choosenBot == "Håkan" && this.$store.state.winner == true) {
          this.$store.commit('updateSessionScore', Math.floor((this.$store.state.numOfGuesses * 3) + 20));
        } else if (this.$store.state.choosenBot == "Miriam" && this.$store.state.winner == true){
           this.$store.commit('updateSessionScore', Math.floor((this.$store.state.numOfGuesses * 4) + 30));
        }
    },
    ranNumBot() {
        var min = this.$store.state.theAnswer - this.minGuess;
        var max = this.$store.state.theAnswer + this.maxGuess;
        var rng = Math.floor(Math.random() * (max - min)) + min;
        this.disableInput();
        if (rng !== this.$store.state.theAnswer) {
          if (this.botGuesses.includes(rng) !== true ) {
            this.botGuesses.push(rng);
            this.$store.state.bot = rng;
        } else {
          if (this.botGuesses.length !== (max - min)) {
            this.ranNumBot();
          }
        }
        } else {
          this.playersTurn = false;
          this.$store.state.winner = false;
          this.$store.state.botWins = true;
          this.$store.state.timerIsOut = false;
          this.stop();
          this.$router.push({ path: 'Winner' });
        }
    },
    makeGuess(value, number, bot) {
      this.ranNumBot();
      this.stopDisable();
      var input = document.getElementById("guess");
      if(this.value < this.$store.state.theAnswer){
        this.$store.state.numOfGuesses++;
        this.playersTurn = false;
        this.stop();
        this.reset();
        this.guessHigher = true;
        this.flash('Guess Higher', 'error', {
          timeout: 2000,
          important: true
        });
        input.value = "";
      }
      else if (this.value > this.$store.state.theAnswer){
        this.$store.state.numOfGuesses++;
        this.playersTurn = false;
        this.stop();
        this.reset();
        this.start();
        this.guessLower = true;
        this.flash('Guess Lower', 'error', {
          timeout: 2000,
          important: true
        });
        input.value = "";
      }
      else if (this.value == this.$store.state.theAnswer){
        this.$store.state.numOfGuesses++;
        this.playersTurn = false;
        //when bot wins
        this.$store.state.winner = true;
        this.$store.state.botWins = false;
        this.show = false;
        this.stop();
        this.$router.push({ path: 'Winner' });
      } 
    },
      OnlyNumbers(e) {
      var keyCode = e.which;
      var ret = ((keyCode >= 48 && keyCode <= 57) || (keyCode >= 8 && keyCode <= 13) || this.numberKeys.indexOf(keyCode) != -1);
      document.getElementById("errormess").style.display = ret ? "none" : "inline";
      return ret;
    },
     disableInput() {
        setTimeout(() => {
        document.getElementById("guess").disabled = true;
        }, 0);
     },
     stopDisable() {
       if (this.$store.state.bot !== this.$store.state.theAnswer) {
         setTimeout(() => {
         document.getElementById("guess").disabled = false;
          this.playersTurn = true;
          this.reset();
          this.start();
          this.setFocus();
          this.guessHigher = false;
          this.guessLower = false;
       }, 2500);
       }
     }
    },
    mounted: function() {
      this.decideMinMax();
    },
    beforeDestroy: function() {
      this.stop();
      this.calculateScore();
    },beforeMount: function() {
      if (this.theQuestion == "") {
      this.$router.push({ path: 'Home' })
      };
    }
  };
</script>

