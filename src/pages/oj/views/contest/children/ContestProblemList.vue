<template key="contestProblem">
<div>
  <div class="problemBox">
    <div class="problemTitle">
      <p>{{$t('m.Problems_List')}}</p>
    </div>
    <div class="problemContent">
      <table v-if="problems.length !== 0" style="text-align: center;">
        <thead>
          <th>{{ $t('m.Th_Problem_Id') }}</th>
          <th>{{ $t('m.Th_Problem_Title') }}</th>
          <th>{{ $t('m.Th_Problem_Difficulty') }}</th>
          <th>{{ $t('m.Th_Problem_Total_Score') }}</th>
          <th>{{ $t('m.Th_Problem_AC_Rate') }}</th>
        </thead>
        <tbody>
          <tr v-for="problem in problems" @click="goContestProblem(problem._id)">
            <td>{{problem._id}}</td>
            <td style="display: flex; flex-direction: column; height: fit-content;">
              <span>{{problem.title}}</span>
              <div style="display: flex;">
                <FieldCategoryBox :boxType="true" :value="FIELD_MAP[problem.field].value"
                                  :boxColor="FIELD_MAP[problem.field].boxColor" />
                <template v-for="(category, idx) in problem.tags">
                  <FieldCategoryBox :boxType="false" :value="'#' + category" :boxColor="'#ffffff'"/>
                </template>
              </div>
            </td>
            <td>{{DIFFICULTY_MAP[problem.difficulty].value}}</td>
            <td>{{problem.total_score}}</td>
            <td>{{getACRate(problem.accepted_number, problem.submission_number)}}</td>
          </tr>
        </tbody>
      </table>
      <div v-else style="text-align: center; font-size: 16px;">
        {{$t('m.No_Problems')}}
      </div>
    </div>
  </div>
  <div>
    <Panel>
      <div slot="title">{{$t('m.Problems_List')}}</div>
      <Table v-if="contestRuleType == 'ACM' || OIContestRealTimePermission"
             :columns="ACMTableColumns"
             :data="problems"
             @on-row-click="goContestProblem"
             :no-data-text="$t('m.No_Problems')"></Table>
      <Table v-else
             :data="problems"
             :columns="OITableColumns"
             @on-row-click="goContestProblem"
             no-data-text="$t('m.No_Problems')"></Table>
    </Panel>
  </div>
</div>
</template>

<script>
  import {mapState, mapGetters} from 'vuex'
  import {ProblemMixin} from '@oj/components/mixins'
  import {DIFFICULTY_MAP, FIELD_MAP} from "../../../../../utils/constants";
  import FieldCategoryBox from "../../../components/FieldCategoryBox.vue";

  export default {
    name: 'ContestProblemList',
    components: {FieldCategoryBox},
    mixins: [ProblemMixin],
    data () {
      return {
        ACMTableColumns: [
          {
            title: '#',
            key: '_id',
            sortType: 'asc',
            width: 150
          },
          {
            title: this.$i18n.t('m.Title'),
            key: 'title'
          },
          {
            title: this.$i18n.t('m.Total'),
            key: 'submission_number'
          },
          {
            title: this.$i18n.t('m.AC_Rate'),
            render: (h, params) => {
              return h('span', this.getACRate(params.row.accepted_number, params.row.submission_number))
            }
          }
        ],
        OITableColumns: [
          {
            title: '#',
            key: '_id',
            width: 150
          },
          {
            title: this.$i18n.t('m.Title'),
            key: 'title'
          }
        ]
      }
    },
    mounted () {
      this.getContestProblems()
    },
    methods: {
      getContestProblems () {
        this.$store.dispatch('getContestProblems').then(res => {
          if (this.isAuthenticated) {
            if (this.contestRuleType === 'ACM') {
              this.addStatusColumn(this.ACMTableColumns, res.data.data)
            } else if (this.OIContestRealTimePermission) {
              this.addStatusColumn(this.ACMTableColumns, res.data.data)
            }
          }
        })
      },
      goContestProblem (id) {
        this.$router.push({
          name: 'contest-problem-details',
          params: {
            contestID: this.$route.params.contestID,
            problemID: id
          }
        })
      }
    },
    computed: {
      ...mapState({
        problems: state => state.contest.contestProblems
      }),
      ...mapGetters(['isAuthenticated', 'contestRuleType', 'OIContestRealTimePermission']),
      DIFFICULTY_MAP() {
        return DIFFICULTY_MAP
      },
      FIELD_MAP() {
        return FIELD_MAP
      },
    }
  }
</script>

<style scoped lang="less">
.problemBox {
  border: 1px solid #e9ece9;
  display: flex;
  flex-direction: column;
  gap: 20px;
  background: var(--box-background-color);
  padding: 15px 20px;
  border-radius: 7px;
}
.problemTitle {
  p {
    text-decoration: none;
    font-size: 24px;
    font-weight: bold;
  }
}
</style>
