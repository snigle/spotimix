<template>
  <div>
    {{ playlist_name }} ({{ alreadyListenedTracks.length }}/{{
      alreadyListenedTracks.length + tracksToListen.length
    }})
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

class Props {
  spotimix = prop<string[]>({ default: [] });
  playlist_id = prop<string>({ default: "", required: true });
  playlist_name = prop<string>({ default: "", required: true });
  access_token = prop<string>({ default: "", required: true });
  playing_track = prop<string>({ default: "" });
}

@Options({
  components: {},
  watch: { access_token: "setAccessToken" },
})
export default class Libeay extends Vue.with(Props) {
  loading = true;

  tracksRaw: PlaylistItem[] = [];

  spotifyApi = new SpotifyWebApi();

  setAccessToken() {
    this.spotifyApi.setAccessToken(this.access_token);
  }

  async mounted() {
    this.loading = true;
    if (!this.playlist_id || !this.access_token) {
      return;
    }
    this.spotifyApi.setAccessToken(this.access_token);

    let offset = 0;
    const limit = 100;
    let items: GetPlaylistItemsResponse;
    do {
      items = await this.spotifyApi.playlists.getPlaylistItems(
        this.playlist_id,
        { offset, limit }
      );
      this.tracksRaw = this.tracksRaw.concat(items.items);
      offset = this.tracksRaw.length;
    } while (items.items.length === limit);
  }

  get isActive(): boolean {
    return (
      this.tracksRaw.find((t) => t.track.uri === this.playing_track) !==
      undefined
    );
  }

  shuffle(array: string[]): string[] {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
  }

  get tracksToListen(): string[] {
    return this.shuffle(
      this.tracksRaw
        .map((t) => t.track.uri)
        .filter((t) => !this.alreadyListenedTracks.includes(t))
    );
  }

  get alreadyListenedTracks(): string[] {
    if (!this.spotimix) {
      return [];
    }

    const currentIndex = this.spotimix.findIndex(
      (uri) => uri === this.playing_track
    );
    if (currentIndex === -1) {
      console.log("track not found in spotimix");
      return [];
    }

    return this.spotimix
      .slice(0, currentIndex + 1)
      .filter((t) => this.tracksRaw.find((tr) => tr.track.uri === t));
  }
}
</script>
