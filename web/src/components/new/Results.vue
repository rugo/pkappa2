<template>
  <div>
    <v-toolbar
      dense
      flat
      id="toolbarReal"
      :style="{
        position: 'fixed',
        width: `${toolbarWidth}px`,
        zIndex: 5,
        borderBottom: '1px solid #eee !important',
      }"
    >
      <v-tooltip bottom>
        <template #activator="{ on, attrs }">
          <v-btn
            v-bind="attrs"
            v-on="on"
            icon
            @click="checkboxAction"
            :disabled="
              streams.result == null || streams.result.Results.length == 0
            "
          >
            <v-icon
              >mdi-{{
                noneSelected
                  ? "checkbox-blank-outline"
                  : allSelected
                  ? "checkbox-marked"
                  : "minus-box"
              }}</v-icon
            >
          </v-btn>
        </template>
        <span>Select</span>
      </v-tooltip>
      <div v-if="noneSelected">
        <v-tooltip bottom>
          <template #activator="{ on, attrs }">
            <v-btn v-bind="attrs" v-on="on" icon @click="fetchStreams">
              <v-icon>mdi-refresh</v-icon>
            </v-btn>
          </template>
          <span>Refresh</span>
        </v-tooltip>
      </div>
      <div v-else>
        <v-menu offset-y right bottom
          ><template #activator="{ on: onMenu, attrs }">
            <v-tooltip bottom>
              <template #activator="{ on: onTooltip }">
                <v-btn v-bind="attrs" v-on="{ ...onMenu, ...onTooltip }" icon>
                  <v-icon>mdi-checkbox-multiple-outline</v-icon>
                </v-btn>
              </template>
              <span>Marks</span>
            </v-tooltip>
          </template>
          <v-list dense>
            <template v-for="tag of groupedTags.mark">
              <v-list-item
                :key="tag.Name"
                link
                @click="
                  markSelectedStreams(
                    tag.Name,
                    tagStatusForSelection[tag.Name] !== true
                  )
                "
              >
                <v-list-item-action>
                  <v-icon
                    >mdi-{{
                      tagStatusForSelection[tag.Name] === true
                        ? "checkbox-outline"
                        : tagStatusForSelection[tag.Name] === false
                        ? "minus-box"
                        : "checkbox-blank-outline"
                    }}</v-icon
                  >
                </v-list-item-action>
                <v-list-item-content>
                  <v-list-item-title>{{
                    tag.Name | tagify("name")
                  }}</v-list-item-title>
                </v-list-item-content>
              </v-list-item>
            </template>
            <v-divider />
            <v-list-item link @click="createMarkFromSelection">
              <v-list-item-action />
              <v-list-item-title>Create new</v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>
      </div>
      <v-spacer />
      <div
        v-if="
          !streams.running &&
          !streams.error &&
          streams.result &&
          streams.result.Results.length != 0
        "
      >
        <span class="text-caption"
          >{{ streams.result.Offset + 1 }}–{{
            streams.result.Offset + streams.result.Results.length
          }}
          of
          {{
            streams.result.MoreResults
              ? "many"
              : streams.result.Results.length + streams.result.Offset
          }}</span
        >
        <v-tooltip bottom>
          <template #activator="{ on, attrs }">
            <v-btn
              v-bind="attrs"
              v-on="on"
              icon
              :disabled="streams.page == 0"
              @click="
                $router.push({
                  name: 'search',
                  query: {
                    q: $route.query.q,
                    p: ($route.query.p | 0) - 1,
                  },
                })
              "
            >
              <v-icon>mdi-chevron-left</v-icon>
            </v-btn>
          </template>
          <span>Previous Page</span>
        </v-tooltip>
        <v-tooltip bottom>
          <template #activator="{ on, attrs }">
            <v-btn
              v-bind="attrs"
              v-on="on"
              icon
              :disabled="!streams.result.MoreResults"
              @click="
                $router.push({
                  name: 'search',
                  query: {
                    q: $route.query.q,
                    p: ($route.query.p | 0) + 1,
                  },
                })
              "
            >
              <v-icon>mdi-chevron-right</v-icon>
            </v-btn>
          </template>
          <span>Next Page</span>
        </v-tooltip>
      </div>
    </v-toolbar>
    <div
      id="toolbarDummy"
      :style="{
        visibility: 'hidden',
        height: `${toolbarHeight}px`,
      }"
    />
    <v-skeleton-loader
      type="table-thead, table-tbody"
      v-if="streams.running || (!streams.result && !streams.error)"
    ></v-skeleton-loader>
    <v-alert type="error" dense v-else-if="streams.error">{{
      streams.error
    }}</v-alert>
    <center v-else-if="streams.result.Results.length == 0">
      <v-icon>mdi-magnify</v-icon
      ><span class="text-subtitle-1">No streams matched your search.</span>
    </center>
    <v-simple-table dense v-else>
      <template #default>
        <thead>
          <tr>
            <th style="width: 0" class="pr-0"></th>
            <th class="text-left pl-0">Tags</th>
            <th class="text-left">Client</th>
            <th class="text-left">Bytes</th>
            <th class="text-left">Server</th>
            <th class="text-left">Bytes</th>
            <th class="text-right">Time</th>
          </tr>
        </thead>
        <tbody>
          <router-link
            v-for="(stream, index) in streams.result.Results"
            :key="index"
            :to="{
              name: 'stream',
              query: { q: $route.query.q, p: $route.query.p },
              params: { streamId: stream.Stream.ID },
            }"
            custom
            #default="{ navigate }"
            style="cursor: pointer"
            :class="{ blue: selected[index], 'lighten-5': selected[index] }"
          >
            <tr @click="navigate" @keypress.enter="navigate" role="link">
              <td style="width: 0" class="pr-0">
                <v-simple-checkbox
                  v-model="selected[index]"
                ></v-simple-checkbox>
              </td>
              <td class="pl-0">
                <template v-for="tag in stream.Tags"
                  ><v-hover v-slot="{ hover }" :key="tag"
                    ><v-chip small
                      ><template v-if="hover"
                        >{{ tag | tagify("type") | capitalize }}
                        {{ tag | tagify("name") }}</template
                      ><template v-else>{{
                        tag | tagify("name")
                      }}</template></v-chip
                    ></v-hover
                  ></template
                >
              </td>
              <td>
                {{ stream.Stream.Client.Host }}:{{ stream.Stream.Client.Port }}
              </td>
              <td>{{ stream.Stream.Client.Bytes }}</td>
              <td>
                {{ stream.Stream.Server.Host }}:{{ stream.Stream.Server.Port }}
              </td>
              <td>{{ stream.Stream.Server.Bytes }}</td>
              <td class="text-right">{{ stream.Stream.FirstPacket }}</td>
            </tr>
          </router-link>
        </tbody>
      </template>
    </v-simple-table>
  </div>
</template>

<style>
</style>

<script>
import { EventBus } from "./EventBus";
import { mapActions, mapGetters, mapState } from "vuex";

export default {
  name: "Results",
  data() {
    return {
      selected: [],
      toolbarWidth: 0,
      toolbarHeight: 0,
      toolbarResizeObserver: null,
    };
  },
  mounted() {
    this.toolbarResizeObserver = new ResizeObserver(this.onToolbarResize);
    this.toolbarResizeObserver.observe(document.getElementById("toolbarReal"));
    this.toolbarResizeObserver.observe(document.getElementById("toolbarDummy"));
    this.onToolbarResize();
    this.fetchStreams();

    const handle = (e, pageOffset) => {
      if (pageOffset >= 1 && !this.streams.result.MoreResults) return;
      let p = this.$route.query.p | 0;
      p += pageOffset;
      if (p < 0) return;
      e.preventDefault();
      this.$router.push({
        name: "search",
        query: { q: this.$route.query.q, p },
      });
    };
    const handlers = {
      j: (e) => {
        handle(e, -1);
      },
      k: (e) => {
        handle(e, 1);
      },
    };
    this._keyListener = function (e) {
      if (["input", "textarea"].includes(e.target.tagName.toLowerCase()))
        return;

      if (!Object.keys(handlers).includes(e.key)) return;
      handlers[e.key](e);
    }.bind(this);
    window.addEventListener("keydown", this._keyListener);
  },
  beforeDestroy() {
    window.removeEventListener("keydown", this._keyListener);
  },
  computed: {
    ...mapState(["streams"]),
    ...mapGetters(["groupedTags"]),
    selectedCount() {
      return this.selected.filter((i) => i === true).length;
    },
    noneSelected() {
      return this.selectedCount == 0;
    },
    allSelected() {
      if (this.selectedCount == 0) return false;
      return this.selectedCount == this.streams.result.Results.length;
    },
    selectedStreams() {
      let res = [];
      for (const [index, value] of Object.entries(this.selected)) {
        if (value) res.push(this.streams.result.Results[index]);
      }
      return res;
    },
    tagStatusForSelection() {
      let res = {};
      for (const s of this.selectedStreams) {
        for (const t of s.Tags) {
          if (!(t in res)) res[t] = 0;
          res[t]++;
        }
      }
      for (const [k, v] of Object.entries(res)) {
        res[k] = v == this.selectedStreams.length;
      }
      return res;
    },
  },
  methods: {
    ...mapActions(["searchStreamsNew", "markTagAdd", "markTagDel"]),
    onToolbarResize() {
      const tbd = document.getElementById("toolbarDummy");
      if (tbd) this.toolbarWidth = tbd.offsetWidth;
      const tbr = document.getElementById("toolbarReal");
      if (tbr) this.toolbarHeight = tbr.offsetHeight;
    },
    checkboxAction() {
      let tmp = [];
      const v = this.noneSelected;
      for (let i = 0; i < this.streams.result.Results.length; i++) tmp[i] = v;
      this.selected = tmp;
    },
    fetchStreams() {
      this.searchStreamsNew({
        query: this.$route.query.q,
        page: this.$route.query.p | 0,
      });
      this.selected = [];
    },
    createMarkFromSelection() {
      let ids = [];
      for (const s of this.selectedStreams) {
        ids.push(s.Stream.ID);
      }
      EventBus.$emit("showCreateTagDialog", {
        tagType: "mark",
        tagStreams: ids,
      });
    },
    markSelectedStreams(tagId, value) {
      let ids = [];
      for (const s of this.selectedStreams) {
        ids.push(s.Stream.ID);
      }
      if (value)
        this.markTagAdd({ name: tagId, streams: ids }).catch((err) => {
          EventBus.$emit("showError", { message: err });
        });
      else
        this.markTagDel({ name: tagId, streams: ids }).catch((err) => {
          EventBus.$emit("showError", { message: err });
        });
    },
  },
  watch: {
    $route: "fetchStreams",
  },
};
</script>