<template>
  <div>
    <div class="ACMRankBox">
      <div class="ACMRankTitle">
        <p>{{contest.title}}</p>
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
      <div class="ACMRankGraph" v-show="showChart">
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
            <td v-for="problem in contestProblems">{{ rank[problem.id].isSet ? rank[problem.id].ac_time : null }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <Pagination :total="total"
                :page-size.sync="limit"
                :current.sync="page"
                @on-change="getContestRankData"
                @on-page-size-change="getContestRankData(1)"
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
        columns: [
          {
            align: 'center',
            width: 50,
            fixed: 'left',
            render: (h, params) => {
              return h('span', {}, params.index + (this.page - 1) * this.limit + 1)
            }
          },
          {
            title: this.$i18n.t('m.User_User'),
            align: 'center',
            fixed: 'left',
            width: 150,
            render: (h, params) => {
              return h('a', {
                style: {
                  display: 'inline-block',
                  'max-width': '150px'
                },
                on: {
                  click: () => {
                    this.$router.push(
                      {
                        name: 'user-home',
                        query: {username: params.row.user.username}
                      })
                  }
                }
              }, params.row.user.username)
            }
          },
          {
            title: 'AC / ' + this.$i18n.t('m.Total'),
            align: 'center',
            width: 100,
            render: (h, params) => {
              return h('span', {}, [
                h('span', {}, params.row.accepted_number + ' / '),
                h('a', {
                  on: {
                    click: () => {
                      this.$router.push({
                        name: 'contest-submission-list',
                        query: {username: params.row.user.username}
                      })
                    }
                  }
                }, params.row.submission_number)
              ])
            }
          },
          {
            title: this.$i18n.t('m.TotalTime'),
            align: 'center',
            width: 100,
            render: (h, params) => {
              return h('span', this.parseTotalTime(params.row.total_time))
            }
          }
        ],
        dataRank: [],
        myDataRank: [],
        options: {
          title: {
            text: this.$i18n.t('m.Top_10_Teams'),
            left: 'center'
          },
          dataZoom: [
            {
              type: 'inside',
              filterMode: 'none',
              xAxisIndex: [0],
              start: 0,
              end: 100
            }
          ],
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
            data: [],
            formatter: (value) => {
              return utils.breakLongWords(value, 16)
            },
            textStyle: {
              fontSize: 12
            }
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
      this.getContestRankData()

      let params = {
        offset: (this.page - 1) * this.limit,
        limit: this.limit,
        contest_id: this.$route.params.contestID,
        force_refresh: this.forceUpdate ? '1' : '0'
      }
      api.getContestRank(params).then(res => {
        this.applyToTable(res.data.data.results)

        const data = res.data.data.results
        let dataRank = JSON.parse(JSON.stringify(data))

        this.getContestProblems().then((res) => {
          this.addRankData(dataRank, res.data.data)
          this.addTableColumns(res.data.data)
          this.addChartCategory(res.data.data)
        })
      })
    },
    methods: {
      ...mapActions(['getContestProblems']),
      addRankData (dataRank, problems) {
        problems.forEach(problem => {
          dataRank.forEach((rank, idx) => {
            dataRank[idx][problem.id] = {isSet: false, problemId: problem._id};
          })
        })
        dataRank.forEach((rank, i) => {
          let info = rank.submission_info
          Object.keys(info).forEach(problemID => {
            dataRank[i][problemID].ac_time = time.secondFormat(info[problemID].ac_time)
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
          // 提取出已AC题目的时间
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
      applyToTable (data) {
        // deepcopy
        let dataRank = JSON.parse(JSON.stringify(data))
        dataRank.forEach((rank, i) => {
          let info = rank.submission_info
          let cellClass = {}
          Object.keys(info).forEach(problemID => {
            dataRank[i][problemID] = info[problemID]
            dataRank[i][problemID].ac_time = time.secondFormat(dataRank[i][problemID].ac_time)
            let status = info[problemID]
            if (status.is_first_ac) {
              cellClass[problemID] = 'first-ac'
            } else if (status.is_ac) {
              cellClass[problemID] = 'ac'
            } else {
              cellClass[problemID] = 'wa'
            }
          })
          dataRank[i].cellClassName = cellClass
        })
        this.dataRank = dataRank
      },
      addTableColumns (problems) {
        // 根据题目添加table column
        problems.forEach(problem => {
          this.columns.push({
            title: problem._id,
            align: 'center',
            // key: problem.id,
            width: problems.length > 15 ? 80 : null,
            renderHeader: (h, params) => {
              return h('a', {
                'class': {
                  'emphasis': true
                },
                on: {
                  click: () => {
                    this.$router.push({
                      name: 'contest-problem-details',
                      params: {
                        contestID: this.contestID,
                        problemID: problem._id
                      }
                    })
                  }
                }
              }, problem._id)
            },
            render: (h, params) => {
              if (params.row[problem.id]) {
                let status = params.row[problem.id]
                let acTime, errorNumber
                if (status.is_ac) {
                  acTime = h('span', status.ac_time)
                }
                if (status.error_number !== 0) {
                  errorNumber = h('p', '(-' + status.error_number + ')')
                }
                return h('div', [acTime, errorNumber])
              }
            }
          })
        })
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
.ACMRankGraph {
  margin: 0 auto;
  height: 400px;
  width: 98%;
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

  .screen-full {
    margin-right: 8px;
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
