<template>
<div>
  <div class="announcementBox">
    <div class="announcementTitle">
      <p>{{$t('m.Announcement')}}</p>
      <div slot="extra">
        <Button v-if="listVisible" type="info" @click="init" :loading="btnLoading">{{$t('m.Refresh')}}</Button>
        <Button v-else type="ghost" icon="ios-undo" @click="goBack">{{$t('m.Back')}}</Button>
      </div>
    </div>
    <div v-if="listVisible" class="announcementContent">
      <a v-for="announcement in announcements" :key="announcement.id" class="announcementItem" @click="goAnnouncement(announcement)">
        <div class="title">{{announcement.title}}</div>
        <div class="date">{{announcement.create_time | localtime('YYYY-M-D')}}</div>
        <div class="creator">{{announcement.created_by.username}}</div>
      </a>
    </div>

  </div>
  <Panel shadow :padding="10">
    <!-- <div slot="title">
      {{title}}
    </div> -->
    <div slot="extra">
      <Button v-if="listVisible" type="info" @click="init" :loading="btnLoading">{{$t('m.Refresh')}}</Button>
      <Button v-else type="ghost" icon="ios-undo" @click="goBack">{{$t('m.Back')}}</Button>
    </div>

    <transition-group name="announcement-animate" mode="in-out">
      <div class="no-announcement" v-if="!announcements.length" key="no-announcement">
        <p>{{$t('m.No_Announcements')}}</p>
      </div>
      <template v-if="listVisible">
        <ul class="announcements-container" key="list">
          <li v-for="announcement in announcements" :key="announcement.title">
            <div class="flex-container">
              <div class="title"><a class="entry" @click="goAnnouncement(announcement)">
                {{announcement.title}}</a></div>
              <div class="date">{{announcement.create_time | localtime }}</div>
              <div class="creator"> {{$t('m.By')}} {{announcement.created_by.username}}</div>
            </div>
          </li>
        </ul>
      </template>

      <template v-else>
        <div v-katex v-html="announcement.content" key="content" class="content-container markdown-body"></div>
      </template>
    </transition-group>
  </Panel>
</div>
</template>

<script>
  import api from '@oj/api'
  import Pagination from '@oj/components/Pagination'

  export default {
    name: 'Announcement',
    components: {
      Pagination
    },
    data () {
      return {
        limit: 10,
        total: 10,
        btnLoading: false,
        announcements: [],
        announcement: '',
        listVisible: true
      }
    },
    mounted () {
      this.init()
    },
    methods: {
      init () {
        this.getContestAnnouncementList()
      },
      getAnnouncementList (page = 1) {
        this.btnLoading = true
        api.getAnnouncementList((page - 1) * this.limit, this.limit).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data.results
          this.total = res.data.data.total
        }, () => {
          this.btnLoading = false
        })
      },
      getContestAnnouncementList () {
        this.btnLoading = true
        api.getContestAnnouncementList(this.$route.params.contestID).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data
        }, () => {
          this.btnLoading = false
        })
      },
      goAnnouncement (announcement) {
        this.announcement = announcement
        this.listVisible = false
      },
      goBack () {
        this.listVisible = true
        this.announcement = ''
      }
    },
  }
</script>

<style scoped lang="less">
.announcementBox {
  border: 1px solid #e9ece9;
  display: flex;
  flex-direction: column;
  gap: 20px;
  background: var(--box-background-color);
  padding: 15px 20px;
  border-radius: 7px;
}
.announcementTitle {
  display: flex;
  justify-content: space-between;
  p {
    text-decoration: none;
    font-size: 24px;
    font-weight: bold;
  }
}
.announcementContent {
  display: flex;
  flex-direction: column;
}
.announcementItem {
  display: flex;
  color: #495060;
  font-size: 16px;
  margin: 0px 20px;
  padding: 15px;
  border-bottom: 1px solid rgba(187, 187, 187, 0.5);
  &:last-child { border-bottom: none; }
  .title {
    flex: 1 auto;
  }
  .date {
    flex-shrink: 0;
    width: 150px;
    text-align: center;
  }
  .creator {
    flex-shrink: 0;
    width: 150px;
    text-align: center;
  }
}

  .announcements-container {
    margin-top: -10px;
    margin-bottom: 10px;
    li {
      padding-top: 15px;
      list-style: none;
      padding-bottom: 15px;
      margin-left: 20px;
      font-size: 16px;
      border-bottom: 1px solid rgba(187, 187, 187, 0.5);
      &:last-child {
        border-bottom: none;
      }
      .flex-container {
        .title {
          flex: 1 1;
          text-align: left;
          padding-left: 10px;
          a.entry {
            color: #495060;
            &:hover {
              color: #2d8cf0;
              border-bottom: 1px solid #2d8cf0;
            }
          }
        }
        .creator {
          flex: none;
          width: 200px;
          text-align: center;
        }
        .date {
          flex: none;
          width: 200px;
          text-align: center;
        }
      }
    }
  }

  .content-container {
    padding: 0 20px 20px 20px;
  }

  .no-announcement {
    text-align: center;
    font-size: 16px;
  }changeLocale

  .announcement-animate-enter-active {
    animation: fadeIn 1s;
  }
</style>
