<template>
  <CardButton v-on:click="gotoEventPage">
    <table id="main-table">
      <tr>
        <td>
          <img
            v-if="imgSrc != null"
            class="event-image"
            v-bind:src="imgSrc"
            v-bind:alt="eventData.id"
          />
          <img v-else class="event-image" src="@/../public/default-event.jpg" />
        </td>
        <td id="text-col">
          <p id="title">{{ eventData.title }}</p>
          {{ getDateStr() }} <br />
          Organizer: {{ eventData.organizerFirstName }}
          {{ eventData.organizerLastName }}<br />
          Attendees: {{ attendeeString }}<br/>
          Categories: {{ categoryString }}
        </td>
      </tr>
      <tr>
        <td style="text-align: right" v-if="userImg">
          <img class="user-image" v-bind:src="userImg" />
        </td>
        <td style="text-align: right" v-else>
          <img class="user-image" src="@/../public/default-user.jpg" />
        </td>
        <td>{{ organizerName }}</td>
      </tr>
    </table>
  </CardButton>
</template>
<script>
import CardButton from "@/components/other/CardButton.vue";
import api from "@/api";
import router from "@/routes";
import store from "@/store";
import { getPretty } from "@/utils/date";
/**
 * Show the event info in a card
 *
 * Takes eventData prop
 *  - is the full detailed event data object passed from backend
 */
export default {
  components: {
    CardButton,
  },
  props: ["eventData"],
  data: function () {
    return {
      imgSrc: null,
      userImg: null,
      organizerName: "",
      categoryString: "",
      isLoggedInUser: false,
    };
  },
  mounted: function () {
    this.loadEventImage();
    this.loadCategories();
    this.loadOrganizer();
  },
  methods: {
    getDateStr() {
      return getPretty(this.eventData.date);
    },
    async loadCategories() {
      // I know this dumb but I'm on a time crunch here
      let s = "";
      for (let name of await store.getCategoryNames(
        this.eventData.categories
      )) {
        s += name + ", ";
      }
      if (s.length > 2) {
        this.categoryString = s.substring(0, s.length - 2);
      }
    },
    gotoEventPage() {
      router.push({ path: `/events/${this.eventData.id}` });
    },
    loadEventImage() {
      api.events.images.getSafeURL(this.eventData.id).then((res) => {
        this.imgSrc = res;
      });
    },
    loadOrganizer() {
      this.organizerName =
        this.eventData.organizerFirstName +
        " " +
        this.eventData.organizerLastName;
      api.users.images.getSafeURL(this.eventData.organizerId).then((res) => {
        this.userImg = res;
      });
    },
  },
  computed: {
    attendeeString() {
      if (this.eventData.capacity == null) {
        if (this.eventData.attendeeCount != null) {
          return this.eventData.attendeeCount;
        } else {
          return "0";
        }
      } else {
        if (this.eventData.attendeeCount != null) {
          return (
            this.eventData.attendeeCount + " out of " + this.eventData.capacity
          );
        } else {
          return "0 out of " + this.eventData.capacity;
        }
      }
    },
  },
};
</script>
<style scoped>
#main {
  margin-top: 5px;
}
#main-table {
  width: 100%;
  height: 100%;
  vertical-align: top;
}
#text-col {
  width: 100%;
}
#title {
  font-size: 20px;
}
.event-image {
  height: 150px;
  width: 150px;
}
.user-image {
  height: 50px;
  width: 50px;
}
</style>