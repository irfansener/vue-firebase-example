<template lang="pug">
  .Popup.Auth
    .Popup-overlay(@click="close")
    .Popup-container
      .Popup-loading(v-show="loginLoading") Yükleniyor...
      button.button(
      type="button",
      :disabled="loginLoading",
      @click="login('github')") Github ile giriş
      button.button(
      type="button",
      :disabled="loginLoading",
      @click="login('google')") Google ile giriş
      button.button(
      type="button",
      :disabled="loginLoading",
      @click="login('twitter')") Twitter ile giriş
      p.alert(v-show="isAlert") {{ alert }}
</template>

<script>
  import firebase from 'firebase'
  import { mapGetters } from 'vuex'
  import { AUTH, FIRESTORE } from '@/firebase'
  import _ from 'lodash'

  export default {
    name: 'Auth',
    data() {
      return {
        alert: ''
      }
    },
    computed: {
      ...mapGetters('Auth', ['User']),
      isAlert() {
        return this.alert !== ''
      },
      loginLoading() {
        return this.$loading.isLoading('user login')
      }
    },
    methods: {
      login(provider) {
        this.$loading.startLoading('user login')
        AUTH.signInWithPopup(this.newProvider(provider))
          .then(result => {
            // giriş yaptı
            console.log(result.user)
            this.$store.commit('Auth/saveRawData', result.user)
            //
            const refUSERS = FIRESTORE.collection('Users')
            //
            refUSERS
              .doc(result.user.uid)
              .get()
              .then(doc => {
                //
                if (!doc.exists) {
                  console.log(result.user.uid)
                  console.log({
                    rawData: this.User.rawData,
                    hasCard: false
                  })
                  refUSERS.doc(result.user.uid).set({
                    rawData: this.User.rawData,
                    hasCard: false
                  })
                }
                if (_.has(doc.data(), 'cardData')) {
                  this.$router.push({ name: 'Home' })
                } else {
                  this.$router.push({ name: 'Profile' })
                }
              })
          })
          .catch(error => {
            if (error.code === 'auth/account-exists-with-different-credential') {
              // daha önce başka bir hesapla giriş yapmış
              AUTH.fetchProvidersForEmail(error.email).then(providers => {
                this.alert = 'daha önce ' + providers + ' hesabı ile giriş yapılmış.'
              })
            }
          })
          .then(() => {
            this.$loading.endLoading('user login')
          })
      },
      newProvider(provider) {
        let newProvider
        switch (provider) {
          case 'github':
            newProvider = new firebase.auth.GithubAuthProvider()
            break
          case 'twitter':
            newProvider = new firebase.auth.TwitterAuthProvider()
            break
          default:
            newProvider = new firebase.auth.GoogleAuthProvider()
        }
        return newProvider
      },
      close() {
        this.$router.push({ name: 'Home' })
      }
    }
  }
</script>

<style lang="scss">
  .Auth {
    .Popup-container {
      max-width: 300px;

      .button + .button {
        margin-top: 10px;
      }

      .alert {
        margin-top: 20px;
      }
    }
  }
</style>
