<template>
  <v-dialog v-model="visible" width="500">
    <v-card>
      <v-card-title>
        <span class="text-h5"
          >{{ tagType | capitalize }} <code>{{ tagName }}</code> details</span
        >
      </v-card-title>
      <v-card-text v-if="tag != null">
        <!--{ "Name": "asd", "Definition": "id:3", "MatchingCount": 1, "UncertainCount": 0, "Referenced": false }-->
        <v-container>
          <v-row no-gutters>
            <v-col class="text-caption">Matching Streams:</v-col>
            <v-col class="text-caption">Uncertain Streams:</v-col>
            <v-col class="text-caption">Is Referenced:</v-col>
          </v-row>
          <v-row no-gutters>
            <v-col>{{ tag.MatchingCount }}</v-col>
            <v-col>{{ tag.UncertainCount }}</v-col>
            <v-col>{{ tag.Referenced ? "Yes" : "No" }}</v-col>
          </v-row>
          <v-row>
            <v-col class="text-caption">Definition:</v-col>
          </v-row>
          <v-row no-gutters>
            <v-col
              ><code>{{ tag.Definition }}</code></v-col
            >
          </v-row>
        </v-container>
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn text @click="visible = false">Close</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script>
import { EventBus } from "./EventBus";
import { mapState } from "vuex";

export default {
  name: "TagDetailsDialog",
  data() {
    return {
      visible: false,
      tagId: "",
      tagType: "",
      tagName: "",
    };
  },
  created() {
    EventBus.$on("showTagDetailsDialog", this.openDialog);
  },
  computed: {
    ...mapState(["tags"]),
    tag() {
      if (this.tags == null) return null;
      for (const t of this.tags) {
        if (t.Name == this.tagId) return t;
      }
      return null;
    },
  },
  methods: {
    openDialog({ tagId }) {
      console.log("asd", tagId);
      this.tagId = tagId;
      this.tagType = tagId.split("/", 1)[0];
      this.tagName = this.tagId.substr(this.tagType.length + 1);
      this.visible = true;
    },
  },
};
</script>