<template>
  <div>
    <div class="ACMRankBox">
      <div class="ACMRankTitle">
        <p>{{ $t('m.Rank') }}</p>
        <div class="ACMRankTitleIcon">
          <screen-full style="height: 18px; width: 18px;"></screen-full>
          <Poptip trigger="hover" placement="left-start">
            <Icon type="android-settings" size="21"></Icon>
            <div slot="content" id="switches" style="display: flex; flex-direction: column; gap: 10px;">
              <span style="width: full; display: flex; justify-content: space-between; align-items: center;">
                <span>{{$t('m.Chart')}}</span>
                <i-switch v-model="showChart"></i-switch>
              </span>
              <span v-if="isContestAdmin" style="display: flex; justify-content: space-between; gap: 10px; align-items: center;">
                <span>{{$t('m.RealName')}}</span>
                <i-switch v-model="showRealName"></i-switch>
              </span>
              <Button type="primary" size="small" @click="downloadRankCSV">{{$t('m.download_csv')}}</Button>
            </div>
          </Poptip>
        </div>
      </div>
      <div v-if="!myDataRank.length" style="text-align: center; font-size: 16px;">{{$t('m.No_Submissions')}}</div>
      <template v-else>
        <div v-show="showChart">
          <ECharts :options="options" ref="chart" auto-resize></ECharts>
        </div>
        <table class="ACMRankContent">
          <thead>
            <th style="width: 50px;">#</th>
            <th>{{ $t('m.User_User') }}</th>
            <th>{{ $t('m.Solved_Problems') }}</th>
            <th v-for="problem in contestProblems"><a style="color: #6CCBFF;" @click="goProblemPage(problem._id)">{{problem._id}}</a></th>
          </thead>
          <tbody>
            <tr v-for="rank in myDataRank">
              <td>{{rank.idx}}</td>
              <td><a @click="goUserPage(rank.user.username)">{{rank.user.username}}</a></td>
              <td>{{rank.accepted_number}}</td>
              <td v-for="problem in contestProblems">
                <div v-if="rank[problem.id].isSet">
                  <span style="margin: 0px 2px 0px 0px; font-size: 10px; background-color: rgb(128, 128, 128); padding: 2px 4px; border-radius: 5px; color: white;">
                    {{rank[problem.id].ac_time | localtime('MM/DD')}}</span>
                  {{rank[problem.id].ac_time | localtime('hh:mm:ss')}}
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </template>
    </div>
    <Pagination :total="total"
                :page-size.sync="limit"
                :current.sync="page"
                @on-change="updateContestData"
                @on-page-size-change="updateContestData()"
                show-sizer></Pagination>
  </div>
</template>

<script>
  import moment from 'moment'
  import { mapActions } from 'vuex'
  import api from '@oj/api'
  import Pagination from '@oj/components/Pagination'
  import ContestRankMixin from './contestRankMixin'
  import time from '@/utils/time'
  import utils from '@/utils/utils'

  export default {
    name: 'acm-contest-rank',
    components: {
      Pagination
    },
    mixins: [ContestRankMixin],
    data () {
      return {
        total: 0,
        page: 1,
        contestID: '',
        myDataRank: [],
        options: {
          title: {
            text: this.$i18n.t('m.Top_10_Teams'),
            left: 'center'
          },
          toolbox: {
            show: true,
            feature: {
              saveAsImage: {show: true, title: this.$i18n.t('m.save_as_image')}
            },
            right: '5%'
          },
          tooltip: {
            trigger: 'axis',
            axisPointer: {
              type: 'cross',
              axis: 'x'
            }
          },
          legend: {
            orient: 'vertical',
            y: 'center',
            right: 0,
            formatter: (value) => {
              return utils.breakLongWords(value, 16)
            },
          },
          grid: {
            x: 80,
            x2: 200
          },
          xAxis: [{
            type: 'time',
            splitLine: false,
            axisPointer: {
              show: true,
              snap: true
            }
          }],
          yAxis: [
            {
              type: 'category',
              boundaryGap: false,
              data: [0]
            }],
          series: []
        }
      }
    },
    mounted () {
      this.contestID = this.$route.params.contestID
      this.updateContestData()
    },
    methods: {
      ...mapActions(['getContestProblems']),
      updateContestData () {
        let params = {
          offset: (this.page - 1) * this.limit,
          limit: this.limit,
          contest_id: this.$route.params.contestID,
          force_refresh: this.forceUpdate ? '1' : '0'
        }
        api.getContestRank(params).then(res => {
          const data = res.data.data.results
          let dataRank = JSON.parse(JSON.stringify(data))

          if (this.page === 1) {
            this.applyToChart(res.data.data.results.slice(0, 10))
          }

          this.getContestProblems().then((res) => {
            this.addRankData(dataRank, res.data.data)
            this.addChartCategory(res.data.data)
          })
          this.total = res.data.data.total
        })
      },
      addRankData (dataRank, problems) {
        problems.forEach(problem => {
          dataRank.forEach((rank, idx) => {
            dataRank[idx][problem.id] = {isSet: false, problemId: problem._id};
          })
        })
        dataRank.forEach((rank, i) => {
          let info = rank.submission_info
          Object.keys(info).forEach(problemID => {
            dataRank[i][problemID].ac_time = moment(this.contest.start_time).add(info[problemID].ac_time, 'seconds').format()
            dataRank[i][problemID].isSet = true
          })
          dataRank[i].idx = (this.page - 1) * this.limit + i + 1;
        })
        this.myDataRank = dataRank;
      },
      addChartCategory (contestProblems) {
        let category = []
        for (let i = 0; i <= contestProblems.length; ++i) {
          category.push(i)
        }
        this.options.yAxis[0].data = category
      },
      applyToChart (rankData) {
        let [users, seriesData] = [[], []]
        rankData.forEach(rank => {
          users.push(rank.user.username)
          let info = rank.submission_info
          let timeData = []
          Object.keys(info).forEach(problemID => {
            if (info[problemID].is_ac) {
              timeData.push(info[problemID].ac_time)
            }
          })
          timeData.sort((a, b) => {
            return a - b
          })

          let data = []
          data.push([this.contest.start_time, 0])
          // index here can be regarded as stacked accepted number count.
          for (let [index, value] of timeData.entries()) {
            let realTime = moment(this.contest.start_time).add(value, 'seconds').format()
            data.push([realTime, index + 1])
          }
          seriesData.push({
            name: rank.user.username,
            type: 'line',
            data
          })
        })
        this.options.legend.data = users
        this.options.series = seriesData
      },
      goUserPage (username) {
        this.$router.push({
          name: 'user-dashboard',
          params: {username: username}
        })
      },
      goProblemPage (problemId) {
        this.$router.push({
          name: 'contest-problem-details',
          params: {
            contestID: this.contestID,
            problemID: problemId
          }
        })
      },
      parseTotalTime (totalTime) {
        let m = moment.duration(totalTime, 's')
        return [Math.floor(m.asHours()), m.minutes(), m.seconds()].join(':')
      },
      downloadRankCSV () {
        utils.downloadFile(`contest_rank?download_csv=1&contest_id=${this.$route.params.contestID}&force_refrash=${this.forceUpdate ? '1' : '0'}`)
      }
    }
  }
</script>

<style scoped lang="less">
.ACMRankBox {
  border: 1px solid #e9ece9;
  display: flex;
  flex-direction: column;
  gap: 20px;
  background: var(--box-background-color);
  padding: 15px 20px;
  border-radius: 7px;
}
.ACMRankTitle {
  display: flex;
  justify-content: space-between;
  align-items: center;
  p {
    text-decoration: none;
    font-size: 24px;
    font-weight: bold;
  }
  .ACMRankTitleIcon {
    display: flex;
    width: 45px;
    justify-content: space-between;
    align-items: center;
  }
}
.ACMRankContent {
  text-align: center;
  th {
    width: 80px;
    color: #7E7E7E;
    font-size: 1.3em;
    padding-bottom: 10px;
  }
  td {
    border-top: 1px solid rgba(0, 0, 0, 0.1);
    padding: 10px 0px;
  }
  tr {
    font-size: 1.05em;
  }
}
.echarts {
  margin: 20px auto;
  height: 400px;
  width: 98%;
}
#switches {
  p {
    margin-top: 5px;
    &:first-child {
      margin-top: 0;
    }
    span {
      margin-left: 8px;
    }
  }
}
</style>
