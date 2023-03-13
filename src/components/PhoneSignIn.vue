<template>
  <div>
    <div class="column">
      <div ref="firebaseui"></div>
    </div>
    <br />
  </div>
</template>

<script>
import firebase from "firebase/app";
import * as firebaseui from "firebaseui";
import "firebaseui/dist/firebaseui.css";
export default {
  name: "PhoneOnlySignIn",
  mounted() {
    this.ui =
      firebaseui.auth.AuthUI.getInstance() ||
      new firebaseui.auth.AuthUI(firebase.auth());
    this.uiConfig = {
      signInSuccessUrl: window.location.href,
      autoUpgradeAnonymousUsers: true,
      signInOptions: [
        {
          provider: firebase.auth.PhoneAuthProvider.PROVIDER_ID,
          defaultCountry: "GB",
        },
      ],
      callbacks: {
        // signInFailure callback must be provided to handle merge conflicts which
        // occur when an existing credential is linked to an anonymous user.
        signInFailure: function (error) {
          // For merge conflicts, the error.code will be
          // 'firebaseui/anonymous-upgrade-merge-conflict'.
          if (error.code != "firebaseui/anonymous-upgrade-merge-conflict") {
            return Promise.resolve();
          }
          // The credential the user tried to sign in with.
          const cred = error.credential;
          // Copy data from anonymous user to permanent user and delete anonymous
          // user at this stage if required.
          // ...
          // Finish sign-in after data is copied.
          return firebase.auth().signInWithCredential(cred);
        },
      },
    };
    this.ui.start(this.$refs.firebaseui, this.uiConfig);
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
@media (orientation: landscape) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
@media (orientation: portrait) {
  .column {
    display: inline-block;
    width: 100%;
  }
}
.firebaseui-idp-anonymous {
  display: none;
}
</style>