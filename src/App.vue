<template>
  <link
    href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css"
    rel="stylesheet"
    integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC"
    crossorigin="anonymous"
  />

  <iframe v-if="soundVisualizer" src="https://visualizer.ianfox.be/visual.php" allow="microphone" />
  <div v-else class="iframe" />
  <div class="overlay">
    <ul class="nav nav-tabs">
      <li class="nav-item">
        <!-- <a class="nav-link active" aria-current="page" href="#">Active</a> -->
        <button class="nav-link" @click="login()">Login</button>
      </li>
      <li class="nav-item">
        <button
          class="nav-link"
          :class="{ active: menu === 'settings' }"
          @click="menu = 'settings'"
        >
          Settings
        </button>
      </li>
      <li class="nav-item">
        <button
          class="nav-link"
          :class="{ active: menu === 'spotimix' }"
          @click="menu = 'spotimix'"
        >
          Spotimix
        </button>
      </li>
    </ul>

    <div class="flex">
      <div class="flex-initial" :hidden="menu !== 'settings'">
        <form @submit="setCredentials()">
          <input v-model="input_genius" placeholder="genius key"/><br/>
          <input v-model="input_spotify" placeholder="spotify key"/><br/>
          <button type="submit">Save</button><br />
          Sound Visualizer : <input type="checkbox" v-model="soundVisualizer" />
          </form>
        <div class="list-group">
          <div
            class="list-group-item"
            v-for="playlist in playlists"
            :key="playlist.id"
          >
            <div>
              {{ playlist.name }}
              <button
                class="btn btn-success"
                v-if="!selectedPlaylists.find((p) => p.id === playlist.id)"
                @click="selectPlaylist(playlist)"
              >
                Add
              </button>
              <button
                class="btn btn-warning"
                v-else
                @click="unselectPlaylist(playlist)"
              >
                Remove
              </button>
              <!-- <button @click="copyToCloud(playlist)">cloud</button> -->
            </div>
          </div>
        </div>
      </div>
      <div class="flex-initial" :hidden="menu !== 'spotimix'">
        <button class="btn btn-primary self-center" @click="generateSpotimix()">
          Generate Spotimix
        </button>
        <div class="list-group">
          <draggable
            v-model="selectedPlaylists"
            class="dragArea list-group w-full"
            item-key="id"
          >
            <template #item="{ element }">
              <div
                class="playlist list-group-item"
                :class="{
                  'list-group-item-default': !playlistRefsVue[element.id]  || !playlistRefsVue[element.id].isActive,
                  'list-group-item-success':
                    playlistRefsVue[element.id] &&
                    playlistRefsVue[element.id].isActive,
                }"
              >
                <playlist
                  :ref="(el) => el && (playlistRefsVue[element.id] = el)"
                  :spotimix="spotimixTracks"
                  :playlist_id="element.id"
                  :playlist_name="element.name"
                  :access_token="spotifyApi.getAccessToken()"
                  :playing_track="playing_track"
                />
                <input
                  class="form-control form-control-sm"
                  type="number"
                  min="0"
                  v-model.number="playlistRepartition[element.id]"
                  @change="savePlaylistRepartition"
                /></div
            ></template>
          </draggable>
        </div>
      </div>
      <div class="flex-auto">
        <player :access_token="access_token" @next="setPlayingTrack()"/>
        <lyrics v-if="currentPlayingTrack"
          :trackName="currentPlayingTrack.name"
          :artist="currentPlayingTrack.artists[0].name"
          :porcentElapsed="this.porcentElapsed"
          :geniusKey="this.geniusKey"
        />
      </div>
      <div class="flex-initial">
        <div class="list-group">
          <div
            class="list-group-item"
            v-for="track in trackPreview.slice(0, 15)"
            :key="track.id"
          >
            <div>
              <img class="float-left" :src="getTrackImage(track)" />
              {{ track.name }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";
import { SpotifyWebApi } from "spotify-web-api-ts";
import _ from "lodash";
import {
  CurrentlyPlaying,
  Playlist,
  PlaylistItem,
  PrivateUser,
  SimplifiedPlaylist,
  Track,
} from "spotify-web-api-ts/types/types/SpotifyObjects";
import PlaylistVue from "./components/Playlist.vue";
import LyricsVue from "./components/Lyrics.vue";
import PlayerVue from "./components/Player.vue";
import { ref } from "@vue/reactivity";
import draggable from "vuedraggable";
import { GetPlaylistItemsResponse } from "spotify-web-api-ts/types/types/SpotifyResponses";
import moment from "moment";

const client_id = "adc341012eea48f581d67ff19d55b250";
const url = new URL(location.href);
const redirect_uri = url.protocol + "//" + url.hostname + url.pathname;
const scopes =
  "user-read-private user-read-email user-library-read playlist-modify-public playlist-modify-private playlist-read-private playlist-read-collaborative user-library-modify user-library-read user-read-recently-played user-top-read user-read-playback-position user-top-read user-read-playback-position user-read-playback-state user-modify-playback-state user-read-currently-playing app-remote-control streaming";

interface TrackDuration {
  startTime: moment.Moment;
  duration: number;
}

@Options({
  components: {
    playlist: PlaylistVue,
    draggable: draggable,
    lyrics: LyricsVue,
    player: PlayerVue,
  },
  watch: {
    playlistRepartition: "savePlaylistRepartition",
    selectedPlaylists: "saveSelectedPlaylists",
    soundVisualizer: "saveSoundVisualizer",
  },
})
export default class App extends Vue {
  input_genius = localStorage.getItem("genius_key") || "";
  input_spotify = localStorage.getItem("spotify_key") || "";
  soundVisualizer = localStorage.getItem("sound_visualizer") === "true";

  spotifyApi = new SpotifyWebApi({
    clientSecret: this.input_spotify,
    clientId: client_id,
    redirectUri: redirect_uri,
  });
  geniusKey = this.input_genius;
  access_token = "";

  playlists: SimplifiedPlaylist[] = [];
  spotimix: Track[] = [];
  spotimixId = "";
  user?: PrivateUser;
  playing_track = "";
  menu = "settings";
  tracksImage: { [key: string]: string } = {};

  trackDuration: TrackDuration = { startTime: moment(), duration: 0 };
  porcentElapsed = 0;

  //Refs
  playlistRefsVue: { [key: string]: PlaylistVue } = {};

  // Inputs
  playlistRepartition: { [key: string]: number } = {};

  setCredentials() {
    this.spotifyApi = new SpotifyWebApi({
      clientSecret: this.input_spotify,
      clientId: client_id,
      redirectUri: redirect_uri,
    });
    localStorage.setItem("genius_key", this.input_genius);
    localStorage.setItem("spotify_key", this.input_spotify);
    this.mounted();
  }

  saveSoundVisualizer() {
    localStorage.setItem("sound_visualizer", `${this.soundVisualizer}`);
  }

  savePlaylistRepartition() {
    console.log("save repartition");
    localStorage.setItem(
      "playlistRepartition",
      JSON.stringify(this.playlistRepartition)
    );
  }
  initPlaylistRepartition() {
    this.playlistRepartition = JSON.parse(
      localStorage.getItem("playlistRepartition") || "{}"
    );
  }

  getTrackImage(track: Track) {
    return track.album.images.find((i) => i.width === 64)?.url;
  }

  updatePorcentElapsed() {
    // console.log("total duration", this.trackDuration.duration)
    // console.log("elapsed ms", moment.duration(moment().diff(this.trackDuration.startTime)).asMilliseconds())
    // console.log("start", this.trackDuration.startTime)
    // console.log("now", moment())
    this.porcentElapsed = Math.round(
      (moment
        .duration(moment().diff(this.trackDuration.startTime))
        .asMilliseconds() /
        (this.trackDuration.duration || 1)) *
        100
    );
    // console.log("porcent", this.porcentElapsed )
    setTimeout(() => this.updatePorcentElapsed(), 1000);
  }

  get trackPreview(): Track[] {
    return this.currentPlayingTrack
      ? [this.currentPlayingTrack, ...this.tracksAfter]
      : this.tracksAfter;
  }

  get spotimixTracks(): string[] {
    return this.spotimix.map((s) => s.uri);
  }

  get currentPlayingTrack(): Track | undefined {
    return this.spotimix.find((t) => t.uri === this.playing_track);
  }

  get refreshToken(): string | null {
    return localStorage.getItem("refresh_token");
  }

  set refreshToken(refreshToken: string | null) {
    if (refreshToken) {
      localStorage.setItem("refresh_token", refreshToken);
    } else {
      localStorage.removeItem("refresh_token");
    }
  }

  saveSelectedPlaylists() {
    console.log("save selected");
    localStorage.setItem(
      "selected_playlists",
      JSON.stringify(this.selectedPlaylists)
    );
  }

  initSelectedPlaylists() {
    this.selectedPlaylists = JSON.parse(
      localStorage.getItem("selected_playlists") || "[]"
    );
  }

  selectedPlaylists: SimplifiedPlaylist[] = [];

  selectPlaylist(playlist: SimplifiedPlaylist) {
    this.selectedPlaylists = _.uniqBy(
      this.selectedPlaylists.concat(playlist),
      (p) => p.id
    );
  }

  unselectPlaylist(playlist: SimplifiedPlaylist) {
    this.selectedPlaylists = this.selectedPlaylists.filter(
      (p) => p.id !== playlist.id
    );
  }

  async updateAccessToken() {
    if (!this.refreshToken) {
      return;
    }
    const accessToken = await this.spotifyApi.getRefreshedAccessToken(
      this.refreshToken
    );
    this.spotifyApi.setAccessToken(accessToken.access_token);
    this.access_token = accessToken.access_token;
    setTimeout(
      () => this.updateAccessToken(),
      accessToken.expires_in * 1000 - 10 * 60 * 1000
    );
  }

  async mounted() {
    const code = new URLSearchParams(window.location.search).get("code");
    if (code) {
      const accessToken = await this.spotifyApi.getRefreshableUserTokens(code);
      this.refreshToken = accessToken.refresh_token;
      window.location.search = "";
    }
    if (this.refreshToken) {
      await this.updateAccessToken();
    } else {
      this.login();
    }

    this.initPlaylistRepartition();
    this.initSelectedPlaylists();

    if (this.selectedPlaylists.length) {
      this.menu = "spotimix";
    }

    this.user = await this.spotifyApi.users.getMe();

    this.playlists = await this.spotifyApi.playlists
      .getMyPlaylists({ limit: 50 })
      .then((lists) => lists.items);

    const spotimixExist = this.playlists.find(
      (playlist) => playlist.name === "spotimix"
    );
    let spotimix: Playlist;
    if (spotimixExist) {
      spotimix = await this.spotifyApi.playlists.getPlaylist(spotimixExist.id);
    } else {
      spotimix = await this.spotifyApi.playlists.createPlaylist(
        this.user.id,
        "spotimix",
        {
          public: false,
          collaborative: false,
          description: "generated playlist by spotimix application",
        }
      );
    }
    this.spotimixId = spotimix.id;

    // Spotimix need to be loaded before setting playingTrack to get the total duration_ms of the song
    this.loadSpotimix();

    this.setPlayingTrack();
    this.updatePorcentElapsed();
  }

  async setPlayingTrack() {
    const current_playing =
      (await this.spotifyApi.player.getCurrentlyPlayingTrack()) as CurrentlyPlaying;
    if (
      !current_playing ||
      current_playing.currently_playing_type !== "track" ||
      !current_playing.item
    ) {
      setTimeout(() => this.setPlayingTrack(), 30 * 1000);
      return;
    }
    this.playing_track = current_playing.item.uri || "";
    const durationBeforeEnd =
      current_playing.item?.duration_ms - (current_playing.progress_ms || 0);

    this.trackDuration = {
      startTime: moment().add(
        -(current_playing.item?.duration_ms || 0) + durationBeforeEnd,
        "milliseconds"
      ),
      duration: current_playing.item?.duration_ms || 0,
    };

    setTimeout(() => this.setPlayingTrack(), durationBeforeEnd + 200);
  }

  login() {
    window.location.href = `https://accounts.spotify.com/authorize?response_type=code&client_id=${client_id}&scope=${encodeURIComponent(
      scopes
    )}&redirect_uri=${encodeURIComponent(redirect_uri)}`;
  }

  async getAllTracksBefore(): Promise<Track[]> {
    if (!this.spotimix) {
      return [];
    }
    const currentTrack =
      (await this.spotifyApi.player.getCurrentlyPlayingTrack()) as CurrentlyPlaying;

    const currentIndex = this.spotimix.findIndex(
      (t) => t.uri === currentTrack.item?.uri
    );
    if (currentIndex === -1) {
      console.log("track not found in spotimix");
      return [];
    }

    return this.spotimix.slice(0, currentIndex + 1);
  }

  get tracksAfter(): Track[] {
    if (!this.spotimix) {
      return [];
    }
    const currentIndex = this.spotimix.findIndex(
      (t) => t.uri === this.playing_track
    );
    if (currentIndex === -1) {
      console.log("track not found in spotimix");
      return [];
    }

    return this.spotimix.slice(currentIndex + 1);
  }

  async loadSpotimix() {
    console.log("load spotimix");
    let offset = 0;
    const limit = 100;
    let items: GetPlaylistItemsResponse;
    let tracksRaw: PlaylistItem[] = [];
    do {
      items = await this.spotifyApi.playlists.getPlaylistItems(
        this.spotimixId,
        { offset, limit }
      );
      tracksRaw = tracksRaw.concat(items.items);
      offset = tracksRaw.length;
    } while (items.items.length === limit);

    this.spotimix = tracksRaw.map((t) => t.track as Track);
  }

  async generateSpotimix() {
    if (!this.spotimix) {
      return;
    }
    await this.loadSpotimix();

    let tracksToAdd: string[] = [];
    // Create stacks to unqueue
    const tracksToListen: { [key: string]: string[] } = {};
    this.selectedPlaylists
      .map((p) => p.id)
      .forEach((playlist_id) => {
        tracksToListen[playlist_id] =
          this.playlistRefsVue[playlist_id].tracksToListen;
      });

    // Take all from stack
    let elementAdded = 0;
    do {
      elementAdded = 0;
      this.selectedPlaylists.forEach((playlist) => {
        const selection = tracksToListen[playlist.id].slice(
          0,
          this.playlistRepartition[playlist.id]
        );
        tracksToAdd.push(...selection);
        tracksToListen[playlist.id] = tracksToListen[playlist.id].slice(
          this.playlistRepartition[playlist.id],
          tracksToListen[playlist.id].length
        );
        elementAdded += selection.length;
        console.log(
          `added ${selection.length} elements on playlist ${
            playlist.name
          }, still ${tracksToListen[playlist.id].length} elements to add`
        );
      });
    } while (elementAdded !== 0);

    // // Get all track before
    const trackPlayed = await this.getAllTracksBefore();
    // Remove allready played
    tracksToAdd = tracksToAdd.filter(
      (t) => !trackPlayed.find((p) => p.uri === t)
    );

    // // Remove all tracks after
    const tracksAfter = this.tracksAfter;
    console.log("tracksAfter", tracksAfter);

    const chunks = _.chunk(tracksAfter, 100);
    for (let i = 0; i < chunks.length; i++) {
      const tracksAfter = chunks[i];
      await this.spotifyApi.playlists.removePlaylistItems(
        this.spotimixId,
        tracksAfter.map((t) => t.uri)
      );
    }

    // Add all tracks after
    const chunks2 = _.chunk(tracksToAdd, 100);
    for (let i = 0; i < chunks2.length; i++) {
      const tracksToAdd = chunks2[i];
      await this.spotifyApi.playlists.addItemsToPlaylist(
        this.spotimixId,
        tracksToAdd.map((t) => t)
      );
    }
  }

  async startPlaylist(playlist: SimplifiedPlaylist) {
    if (!this.spotimix) {
      return;
    }
    this.spotimix = await this.spotifyApi.playlists
      .getPlaylist(this.spotimixId)
      .then((r) => r.tracks.items.map((i) => i.track as Track));

    let tracksToAdd = (
      await this.spotifyApi.playlists.getPlaylistItems(playlist.id)
    ).items;
    // Get all track before
    const trackPlayed = await this.getAllTracksBefore();
    // Remove allready played
    tracksToAdd = tracksToAdd.filter(
      (t) => !trackPlayed.find((p) => p.uri === t.track.uri)
    );

    // Remove all tracks after
    const tracksAfter = this.tracksAfter;

    await this.spotifyApi.playlists.removePlaylistItems(
      this.spotimixId,
      tracksAfter.map((t) => t.uri)
    );
    // Add all tracks after
    await this.spotifyApi.playlists.addItemsToPlaylist(
      this.spotimixId,
      tracksToAdd.map((t) => t.track.uri)
    );
  }

  async copyToCloud(playlist: SimplifiedPlaylist) {
    if (!this.user) {
      return;
    }
    const tracks = await this.spotifyApi.playlists.getPlaylistItems(
      playlist.id
    );
    const cloudName = playlist.name + "-cloud";
    let cloudPlaylistIDTmp: string | undefined = this.playlists.find(
      (p) => p.name === cloudName
    )?.id;
    if (!cloudPlaylistIDTmp) {
      const cloudPlaylist = await this.spotifyApi.playlists.createPlaylist(
        this.user.id,
        cloudName,
        {
          public: playlist.public || false,
          collaborative: playlist.collaborative || false,
          description: playlist.description || undefined,
        }
      );
      cloudPlaylistIDTmp = cloudPlaylist.id;
    }

    const cloudPlaylistID = cloudPlaylistIDTmp;

    const trackIDs = await Promise.all(
      tracks.items.map(async (t) => {
        if (t.is_local) {
          const cloudTrack = await this.spotifyApi.search.searchTracks(
            t.track.name,
            { limit: 1 }
          );
          if (cloudTrack.items[0]) {
            return cloudTrack.items[0].uri;
          } else {
            console.log("fail to find track with name", t.track.name);
            return "";
          }
        } else {
          return t.track.uri;
        }
      })
    );

    this.spotifyApi.playlists.addItemsToPlaylist(
      cloudPlaylistID,
      trackIDs.filter((t) => t)
    );
  }
}
</script>

<style lang="less">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;

  @tailwind base;
  @tailwind components;
  @tailwind utilities;
}

iframe,
.iframe {
  position: absolute;
  width: 100%;
  height: 100%;
  border: none;
  box-sizing: border-box;
}
.iframe {
  background-color: black;
}

.overlay {
  z-index: 100;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  // background-color: #a23d3d38;
  color: white;
}

.playlist div {
  display: inline-block;
}

input[type="number"].form-control {
  display: inline-block;
  width: 50px;
  margin-left: 5px;
  // float:right;
}

.list-group {
  .list-group-item-default {
    background-color: #f9fafb1a;
    color: white;
  }
}
input {
  color: black
}
</style>
