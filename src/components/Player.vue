<template>
  <div>
    <button class="btn btn-success" @click="play()">Play</button>
      <button class="btn btn-primary" @click="pause()">Pause</button>
      <button class="btn btn-warning" @click="next()">Next</button>
  </div>
</template>

<script lang="ts">
import { Options, prop, Vue } from "vue-class-component";
import { SpotifyWebApi } from "spotify-web-api-ts";
import {
  CurrentlyPlaying,
  Playlist,
  PlaylistItem,
  PrivateUser,
  SimplifiedPlaylist,
  Track,
} from "spotify-web-api-ts/types/types/SpotifyObjects";
import { GetPlaylistItemsResponse } from "spotify-web-api-ts/types/types/SpotifyResponses";
import genius from "genius-lyrics";
import axios from "axios";
import _ from "lodash";
import "../assets/rezzo.wav";

class Props {
  access_token = prop<string>({ default: "" });
}

@Options({
  components: {},
  emits: ["next"],
  watch: { access_token: "setAccessToken" },
})
export default class Player extends Vue.with(Props) {
  loading = true;
  spotifyApi = new SpotifyWebApi();


  setAccessToken() {
    this.spotifyApi.setAccessToken(this.access_token);
  }

  play() {
    this.spotifyApi.player.play();
    this.$emit("next");
  }

  pause() {
    this.spotifyApi.player.pause();
  }
  async next() {
    const delay = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))
    const audio = new Audio(require("../assets/rezzo.wav"));
    
    await this.spotifyApi.player.setVolume(80);
    audio.volume = 0.3;
    await audio.play();
    await delay(3000);
    audio.volume = 0.5;
    await this.spotifyApi.player.setVolume(50);

    await delay(1000);
    audio.volume = 1;
    await this.spotifyApi.player.setVolume(30);

    await this.spotifyApi.player.skipToNext();

    await delay(1000);
    audio.volume = 0.5;
    await this.spotifyApi.player.setVolume(50);

    await delay(1000);
    audio.volume = 0.3;
    await this.spotifyApi.player.setVolume(80);

    await delay(3000);
    audio.volume = 0.3;
    await this.spotifyApi.player.setVolume(100);

    await audio.pause();
    this.$emit("next");

  }
}
</script>
