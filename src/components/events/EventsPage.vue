<template>
  <PageContent title="Events">
    <!-- Search bar -->
    <table class="grow">
      <td class="grow">
        <input
          class="grow"
          v-model="filters.searchString"
          type="text"
          placeholder="Search..."
          maxlength="63"
        />
      </td>
      <td>
        <button type="button" v-on:click="search">Search</button>
      </td>
    </table>
    <!-- Other filters -->
    <table class="filter-table">
      <!-- Sort -->
      <table class="grow">
        <td>
          <p style="white-space: nowrap">Sort By:</p>
        </td>
        <td class="grow">
          <select class="grow" name="cars" id="cars" v-model="sortBy">
            <option v-for="s of sortOptions" v-bind:key="s" v-bind:value="s">
              {{ s }}
            </option>
          </select>
        </td>
      </table>
      <!-- Categories -->
      <tr>
        Filter By Categories:
        <input
          v-model="filterByCategories"
          type="checkbox"
          v-on:change="filterCategories"
        />
      </tr>
      <tr v-if="filterByCategories">
        Showing all events with one or more of the selected
        <div class="container">
          <div
            v-for="option in categoryOptions"
            v-bind:key="option.name"
            class="filter-options"
          >
            {{ option.name }}
            <input
              v-model="option.enabled"
              v-on:change="filterCategories"
              type="checkbox"
            />
          </div>
        </div>
      </tr>
    </table>
    <br />
    Showing {{ showingEvents.length }} of {{ events.length }} Results

    <!-- Show the events -->
    <div
      v-for="event in showingEvents"
      v-bind:key="event.eventId"
      class="event"
    >
      <EventsPage_Event v-bind:eventData="event" />
    </div>
    <Pagination
      v-model="pageIndex"
      v-on:change="changePage"
      v-bind:lastPageIndex="lastPageIndex"
    />
  </PageContent>
</template>
<script>
import EventsPage_Event from "@/components/events/EventsPage_EventCard.vue";
import PageContent from "@/components/other/PageContent.vue";
import Pagination from "@/components/other/Pagination.vue";
import store from "@/store";
import api from "@/api";
export default {
  components: {
    EventsPage_Event,
    PageContent,
    Pagination,
  },
  data: function () {
    return {
      filterByCategories: false,
      categoryOptions: [],
      allEvents: [],
      events: [],
      showingEvents: [],
      pageIndex: 0,
      pageSize: 10,
      lastPageIndex: 0,
      filters: {
        searchString: "",
      },
      sortBy: null,
      sortOptions: [
        "Attendees Ascending",
        "Attendees Descending",
        "Date Close",
        "Date Far",
      ],
    };
  },
  mounted() {
    this.loadEvents();
    this.setLastPageIndex();
    this.loadCategories();
  },
  methods: {
    async loadCategories() {
      let data = await store.getCategories();
      let cs = new Array(data.length - 1);
      for (const id in data) {
        let name = data[id];
        cs[id - 1] = {
          name,
          enabled: false,
          id,
        };
      }
      this.categoryOptions = cs;
    },
    changePage() {
      this.pageIndex = Math.min(this.pageIndex, this.lastPageIndex);
      this.showingEvents = this.events.slice(
        this.pageSize * this.pageIndex,
        this.pageSize * (1 + this.pageIndex)
      );
    },
    async sortEvents() {
      // Am being lazy and getting server to sort dates
      await this.loadEvents();
      if (this.sortBy === "Attendees Ascending") {
        this.events.sort((a, b) => {
          return parseInt(a.numAcceptedAttendees) >
            parseInt(b.numAcceptedAttendees)
            ? 1
            : -1;
        });
      } else if (this.sortBy === "Attendees Descending") {
        this.events.sort((a, b) => {
          return parseInt(a.numAcceptedAttendees) <
            parseInt(b.numAcceptedAttendees)
            ? 1
            : -1;
        });
      }
      this.changePage();
    },
    onLoad() {
      this.filterCategories();
      this.changePage();
    },
    async loadEvents() {
      let sort = "DATE_ASC";
      if (this.sortBy === "Date Far") sort = "DATE_DESC";
      let res = await api.events.get({ sortBy: sort });
      let rawEvents = this.filterForTitle(res.data, this.filters.searchString);

      // Load full event data
      let fullEvents = [];
      let responses = 0;
      for (let event of rawEvents) {
        api.events
          .getOne(event.eventId)
          .then((res) => {
            fullEvents.push(res.data);
            responses += 1;

            if (responses == rawEvents.length) {
              this.onAllEventsLoaded(fullEvents);
            }
          })
          .catch(() => {
            responses += 1;
            if (responses == rawEvents.length) {
              this.onAllEventsLoaded(fullEvents);
            }
          });
      }
    },
    onAllEventsLoaded(events) {
      let sorted = [];
      for (let event of events) {
        if (new Date(event.date) >= new Date()) {
          sorted.push(event);
        }
      }
      this.events = sorted;
      this.allEvents = sorted;

      this.onLoad();
    },
    search() {
      this.pageIndex = 0;
      this.setLastPageIndex();
      this.loadEvents();
    },
    setLastPageIndex() {
      this.lastPageIndex = Math.floor((this.events.length - 1) / this.pageSize);
    },
    async filterCategories() {
      if (this.filterByCategories) {
        let es = [];
        for (let e of this.allEvents) {
          let done = false;
          for (let category of this.categoryOptions) {
            if (category.enabled) {
              if (e.categories.includes(parseInt(category.id))) {
                es.push(e);
                done = true;
                break;
              }
            }
          }
          if (done) continue;
        }
        this.events = es;
      } else {
        this.events = this.allEvents;
      }
      this.setLastPageIndex();
      this.changePage();
    },
    filterForTitle(events, q) {
      let es = [];
      for (let e of events) {
        if (e.title.includes(q)) {
          es.push(e);
        }
      }
      return es;
    },
  },
  watch: {
    sortBy() {
      this.sortEvents();
    },
  },
};
</script>
<style scoped>
.event {
  margin: 5px;
}
.search-button {
  background-color: var(--primary-faded);
}
.filter-table {
  width: 100%;
  text-align: left;
  /* border: 1px solid black; */
}
.grow {
  width: 100%;
}
.container {
  display: flex;
  flex-wrap: wrap;
  width: 90%;
  padding-left: 10%;
}
.filter-options {
  white-space: nowrap;
  padding-left: 15px;
}
</style>