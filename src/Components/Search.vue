<template>
  <div id="search-contain">
    <button class="filter-button" @click="toggleFilter">Filter</button>
    <input
      class="query-input"
      type="text"
      v-model="searchQuery"
      @keyup.enter="search"
      :disabled="loading || type === 'my'"
      :placeholder="placeholder"
    /><br />
    <div class="search-condition-contain">
      <button
        class="search-button"
        @click="search"
        v-if="!loading"
        title="Search"
      >
        Search
      </button>
      <button
        class="searching-button"
        v-else
        title="Searching"
        :disabled="true"
      >
        <div class="animated pulse infinite">Searching</div>
      </button>
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";
import { EventBus } from "../main";
import { GET, POST } from "../Utilities/RequestHelper.js";
import escapeStringRegexp from "../Utilities/EscapeRegex.js";

export default {
  name: "Search",
  data() {
    return {
      searchQuery: ""
    };
  },
  methods: {
    emitPicked() {
      EventBus.$emit("pick-ready");
    },
    emitError(error) {
      EventBus.$emit("error-show", error);
    },
    toggleFilter() {
      EventBus.$emit("toggle-filter");
    },
    // Switch cases for search types
    search() {
      if (this.$route.name !== "PlayVideo") {
        this.$router.push("/");
      }

      this.$store.commit("LOADING_ON");
      EventBus.$emit("toggle-filter", false);

      // if(this.type !== "my"){
      //   EventBus.$emit("toggle-search", false);
      // }

      switch (this.type) {
        case "video":
          this.searchVideos(this.searchQuery);
          break;
        case "my":
          if (this.token) {
            this.searchMySubscriptions(this.searchQuery);
          } else {
            this.$store.commit("LOADING_OFF");
            this.emitError("Must be logged in to use this option");
            this.$router.push("auth");
          }
          break;
        case "channel":
          this.searchChannels(this.searchQuery);
          break;
        case "channel-id":
          this.pickChannel(this.searchQuery);
          break;
        case "playlist":
          this.searchPlaylist(this.searchQuery);
          break;
        default:
          break;
      }
    },
    // Set's up new videos and reset vid num
    setVideos(videos) {
      console.log("Videos", videos);
      if (this.filterKeywords) {
        console.log("Filterkeywords", this.filterKeywords);
        videos = videos.filter(video => {
          if (video && video.snippet && video.snippet.title) {
            return !new RegExp(
              escapeStringRegexp(this.filterKeywords.split(/[ ,]+/).join("|")),
              "g"
            ).test(video.snippet.title.toLowerCase());
          } else {
            return true;
          }
        });
      }
      this.$store.commit("UPDATE_VIDEO_LIST", videos);
      this.searchQuery = "";
      EventBus.$emit("toggle-search", false);
      EventBus.$emit("toggle-filter", false);
    },
    // Search videos by keywords
    searchVideos(query) {
      GET(
        `search?part=snippet&q=${query}&maxResults=${this.maxResults}&type=video&order=${this.filterSort}&safeSearch=${this.filterSafe}&key=${process.env.API_KEY}`
      )
        .then(res => {
          this.setVideos(res.data.items);
        })
        .catch(err => {
          this.$store.commit("LOADING_OFF");
          this.emitError("Error finding videos");
        });
    },
    // Get newest videos from subscriptions and massage data
    searchMySubscriptions() {
      this.searchQuery = "";
      let subscriptions;

      // Make sure maxResults is less the 10, if not goto 5
      if (this.maxResults > 10) {
        this.$store.commit("UPDATE_MAX_RESULTS", 5);
      }

      const recursiveSubscription = (nextPage = null) => {
        if (!nextPage) {
          // First Page
          GET(
            `subscriptions?part=snippet&mine=true&maxResults=50&key=${process.env.API_KEY}&access_token=${this.token}`
          )
            .then(res => {
              subscriptions = res.data.items;
              let hasNextPage = res.data.nextPageToken ? true : false;

              if (hasNextPage) {
                let nextPage = res.data.nextPageToken;
                recursiveSubscription(nextPage);
              } else {
                this.massageSubscriptions(subscriptions);
              }
            })
            .catch(err => {
              console.log("First page error", err);
              this.$store.commit("LOADING_OFF");
              if (err.response.status == 401) {
                if (this.token) {
                  this.emitError(
                    "Sign in expired. Need to re authenticate before getting subscriptions"
                  );
                } else {
                  this.emitError(
                    "Need to authenticate before getting subscriptions"
                  );
                }
                this.$router.push("auth");
              }
            });
        } else {
          // Next Pages
          GET(
            `subscriptions?part=snippet&mine=true&pageToken=${nextPage}&maxResults=50&key=${process.env.API_KEY}&access_token=${this.token}`
          )
            .then(res => {
              subscriptions.push(...res.data.items);
              let hasNextPage = res.data.nextPageToken ? true : false;
              if (hasNextPage) {
                let nextPage = res.data.nextPageToken;
                recursiveSubscription(nextPage);
              } else {
                this.massageSubscriptions(subscriptions);
              }
            })
            .catch(err => {
              console.log("recursiveSubscription Next Page Error", err);
              recursiveSubscription(index + 1);
            });
        }
      };

      recursiveSubscription();
    },
    // Go through every subscription and retrieve videos from them
    massageSubscriptions(data) {
      let videos = [];

      const recurseMassage = (index = 0) => {
        GET(
          `search?part=snippet&channelId=${data[index].snippet.resourceId.channelId}&maxResults=${this.maxResults}&type=video&order=date&safeSearch=${this.filterSafe}&key=${process.env.API_KEY}`
        )
          .then(res => {
            videos.push(...res.data.items);
            if (index < data.length - 1) {
              recurseMassage(index + 1);
            } else {
              videos.sort(function(a, b) {
                return a.snippet.publishedAt < b.snippet.publishedAt
                  ? 1
                  : b.snippet.publishedAt < a.snippet.publishedAt
                  ? -1
                  : 0;
              });
              this.setVideos(videos);
            }
          })
          .catch(err => {
            console.log("Massage subscriptions Error", err);
            if (index < data.length - 1) recurseMassage(index + 1);
            else this.setVideos(videos);
          });
      };

      if (data.length > 0) {
        recurseMassage();
      } else {
        this.$store.commit("LOADING_OFF");
        this.emitError("You have no subscriptions");
      }
    },
    // Search channels by username
    searchChannels(query) {
      this.searchQuery = "";
      GET(
        `search?part=snippet&type=channel&q=${query}&order=relevance&maxResults=20&key=${process.env.API_KEY}`
      )
        .then(res => {
          this.$store.commit("LOADING_OFF");
          if (res.data.items.length > 0) {
            this.$store.commit("UPDATE_CHANNELS", res.data.items);
            this.emitPicked();
          } else {
            this.emitError("Could not find any results");
          }
        })
        .catch(err => {
          console.log("Search channels Error", err);
          this.$store.commit("LOADING_OFF");
          this.emitError("Error finding channel");
        });
    },
    // Search playlists by name
    searchPlaylist(query) {
      this.searchQuery = "";
      GET(
        `search?part=snippet&type=playlist&q=${query}&maxResults=20&key=${process.env.API_KEY}`
      )
        .then(res => {
          this.$store.commit("LOADING_OFF");
          if (res.data.items.length > 0) {
            this.$store.commit("UPDATE_PLAYLISTS", res.data.items);
            this.emitPicked();
          } else {
            this.emitError("Could not find any results");
          }
        })
        .catch(err => {
          console.log("Search Playlist Error", err);
          this.$store.commit("LOADING_OFF");
          this.emitError("Error finding channel");
        });
    }
  },
  computed: {
    ...mapState({
      loading: "loading",
      token: "token",
      type: "type",
      maxResults: "maxResults",
      videos: "videos",
      filterSort: "filterSort",
      filterSafe: "filterSafe",
      filterDate: "filterDate",
      filterDuration: "filterDuration",
      filterKeywords: "filterKeywords"
    }),
    placeholder() {
      switch (this.type) {
        case "my":
          return "Newest videos";
          break;
        case "video":
          return "Search by keyword";
          break;
        case "channel":
          return "Search by channel username";
          break;
        case "channel-id":
          return "Search by channel id";
          break;
        case "playlist":
          return "Search by playlist id";
          break;
        default:
          return "Search";
          break;
      }
    }
  }
};
</script>

<style scoped>
#search-contain {
  display: flex;
  justify-content: center;
  height: 40px;
}

.filter-button {
  height: 100%;
  order: 1;
  font-size: 14pt;
}

.query-input {
  width: 20vw;
  text-align: center;
  order: 2;
  font-size: 14pt;
}

.search-condition-contain {
  height: 40px;
  order: 3;
}

.search-button,
.searching-button {
  height: 100%;
  font-size: 14pt;
}

.searching-button {
  cursor: progress;
  color: var(--highlighDark);
  background-color: var(--activeDark);
}

.searching-button:hover {
  color: inherit;
}

/* Mobile */
@media (max-width: 475px) {
  .filter-button {
    order: 2;
  }
  .query-input {
    order: 1;
    width: 40vw;
  }
  .search-condition-contain {
    order: 3;
  }

  .filter-button,
  .search-button {
    padding: 10px 10px;
  }
}
</style>
