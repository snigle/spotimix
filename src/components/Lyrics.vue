<template>
  <div>
      <p class="text-6xl">
        <span v-for="(l, index) in lyrics" :key="index">
          <span :hidden="index < currentIndex-5 || index > currentIndex + 7">{{l}}<br>
          </span>
          </span>
      </p>
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

class Props {
  artist = prop<string>({ default: [] });
  trackName = prop<string>({ default: "", required: true });
  porcentElapsed = prop<number>({default: 0, required: true})
  geniusKey = prop<string>({});
}

@Options({
  components: {},
  watch: {
    trackName: "watchTrackName",
    geniusKey: "watchGeniusKey",
     },
})
export default class Lyrics extends Vue.with(Props) {
  loading = true;
  lyrics : string[] = []

  genius = axios.create({
    baseURL: "https://api.genius.com",
    params: {
      access_token:
        this.geniusKey,
    },
  });

watchGeniusKey() {
this.genius = axios.create({
    baseURL: "https://api.genius.com",
    params: {
      access_token:
        this.geniusKey,
    },
  });
}

get currentIndex(): number {
  if (!this.lyrics) {
    return 0
  }
  const index = Math.round(this.lyrics.length * this.porcentElapsed / 100);
  return index
}

  async watchTrackName() {
    this.lyrics = [];
    console.log("track name changed", this.trackName);
    if (this.trackName == "") {
      return;
    }
    // title: this.track_name,
    // artist: this.artist,
    let search = await this.genius
      .get(`/search`, { params: { q: `${this.artist} ${this.trackName}` } })
      .then((r) => r.data.response?.hits);
    console.log("find", search);
    if (!search || search.length == 0) {
      // Retry without artist
      search = await this.genius
      .get(`/search`, { params: { q: `${this.trackName}` } })
      .then((r) => r.data.response?.hits);
    console.log("find", search);
    }
    if (!search || search.length == 0) {
    return
    }
    const firstResult = search.filter((s: any) => s.type === "song")[0];
    const song = await this.genius
      .get(firstResult.result.api_path)
      .then((r) => r.data.response.song);
    console.log("found song", song);
    const div = document.createElement("div");
    div.innerHTML = song.embed_content
      .replace(`src='//`, `src='${location.protocol}//`)
      .trim();
    // console.log(div.innerHTML);
    // console.log(song.embed_content)

    const scriptURL = div.querySelector("script")?.src || "";
    const scriptTxt = await fetch(scriptURL).then((r) => r.text());

    const jsonline = scriptTxt
      .split("\n")
      .find((l) => l.match(/document.write\(JSON.parse/));
    // console.log("json line", jsonline);
    if (!jsonline) return;
    const jsonPart = jsonline
      .replace(/^.*document.write\(/, "")
      .replace(/\)$/, "");
    // console.log("jsonPart", jsonPart)
    let lyrics = eval(jsonPart);
    // console.log("lyrics = ", lyrics)

    // Only take body
    const span = document.createElement("span");
    span.innerHTML = lyrics;
    // console.log("span", span)
    const blocks = span.querySelector(".rg_embed_body");
    if (blocks) {
      // console.log("blocks", blocks.innerHTML)
      lyrics = blocks.innerHTML
    }

    const lyricsWithoutHTML: string[] = lyrics
      .split("\n")
      .map((l: string) => this.extractContent(l, true));
    const cleanedLyrics = _.reduce(
      lyricsWithoutHTML,
      (prev, current, index) => {
        if (!index) {
          return [current];
        }
        if (prev[prev.length - 1].trim() === "" && current.trim() === "") {
          return prev;
        }
        if (current.match(/\[.*\]/)){
          return prev
        }
        return [...prev, current];
      },
      [] as string[]
    );

    this.lyrics = cleanedLyrics
    div.innerHTML = cleanedLyrics.join("<br>");
  }

  extractContent(s: string, space: boolean) {
    const span = document.createElement("span");
    span.innerHTML = s;
    if (space) {
      const children = span.querySelectorAll("*");
      for (let i = 0; i < children.length; i++) {
        if (children[i].textContent) children[i].textContent += " ";
      }
    }
    return [span.textContent || span.innerText].toString().replace(/ +/g, " ");
  }

  async mounted() {
    console.log("lyrics mounted", this.trackName);
    this.watchTrackName();
  }
}
</script>
