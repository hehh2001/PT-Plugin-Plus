<template>
  <div class="home">
    <v-alert :value="true" type="info">{{ words.title }}</v-alert>
    <v-card>
      <v-card-title>
        <v-btn color="success" @click="getInfos" :loading="loading" :title="words.getInfos">
          <v-icon class="mr-2">cached</v-icon>
          {{words.getInfos}}
        </v-btn>
        <v-btn to="/user-data-timeline" color="success" :title="words.timeline">
          <v-icon>timeline</v-icon>
        </v-btn>

        <v-btn to="/statistic" color="success" :title="words.statistic">
          <v-icon>equalizer</v-icon>
        </v-btn>

        <v-menu :close-on-content-click="false" offset-y>
          <template v-slot:activator="{ on }">
            <v-btn color="blue" dark v-on="on" :title="words.settings">
              <v-icon>settings</v-icon>
            </v-btn>
          </template>

          <v-card>
            <v-container grid-list-xs>
              <v-switch
                color="success"
                v-model="showSiteName"
                :label="words.siteName"
                @change="updateViewOptions"
              ></v-switch>
              <v-switch
                color="success"
                v-model="showUserName"
                :label="words.userName"
                @change="updateViewOptions"
              ></v-switch>
              <v-switch
                color="success"
                v-model="showUserLevel"
                :label="words.userLevel"
                @change="updateViewOptions"
              ></v-switch>

              <v-switch
                color="success"
                v-model="showWeek"
                :label="words.week"
                @change="updateViewOptions"
              ></v-switch>
            </v-container>
          </v-card>
        </v-menu>

        <v-spacer></v-spacer>

        <v-text-field
          class="search"
          v-model="filterKey"
          append-icon="search"
          label="Search"
          single-line
          hide-details
        ></v-text-field>
      </v-card-title>

      <v-data-table
        :search="filterKey"
        :headers="headers"
        :items="sites"
        :pagination.sync="pagination"
        item-key="host"
        class="elevation-1"
        ref="userDataTable"
      >
        <template slot="items" slot-scope="props">
          <!-- 站点 -->
          <td class="center">
            <v-badge color="red messageCount" overlap>
              <template
                v-slot:badge
                v-if="props.item.user.messageCount>0"
                :title="words.newMessage"
              >{{ props.item.user.messageCount>10?"":props.item.user.messageCount }}</template>
              <v-avatar :size="showSiteName ? 18 : 24" @click.stop="getSiteUserInfo(props.item)">
                <img :src="props.item.icon">
              </v-avatar>
            </v-badge>

            <br>
            <a
              :href="props.item.activeURL"
              target="_blank"
              rel="noopener noreferrer nofollow"
              class="nodecoration"
              v-if="showSiteName"
            >
              <span class="caption">{{ props.item.name }}</span>
            </a>
          </td>
          <td>{{ showUserName ? props.item.user.name : "****" }}</td>
          <td>{{ showUserLevel ? props.item.user.levelName : "****" }}</td>
          <td class="number">
            <div>
              {{ props.item.user.uploaded | formatSize }}
              <v-icon small color="green darken-4">expand_less</v-icon>
            </div>
            <div>
              {{ props.item.user.downloaded | formatSize }}
              <v-icon small color="red darken-4">expand_more</v-icon>
            </div>
          </td>
          <td class="number">{{ props.item.user.ratio | formatRatio }}</td>
          <td class="number">{{ props.item.user.seeding }}</td>
          <td class="number">{{ props.item.user.seedingSize | formatSize }}</td>
          <td class="number">{{ props.item.user.bonus | formatNumber }}</td>
          <td class="number">{{ props.item.user.joinTime | timeAgo(showWeek) }}</td>
          <td class="number">
            <v-btn
              depressed
              small
              :to="`statistic/${props.item.host}`"
              :title="words.statistic"
            >{{ props.item.user.lastUpdateTime | formatDate('YYYY-MM-DD HH:mm:ss') }}</v-btn>
          </td>
          <td class="center">
            <v-progress-circular
              indeterminate
              :width="3"
              size="30"
              color="green"
              v-if="props.item.user.isLoading"
            >
              <v-icon
                v-if="props.item.user.isLoading"
                @click="abortRequest(props.item)"
                color="red"
                small
                :title="words.cancelRequest"
              >cancel</v-icon>
            </v-progress-circular>
            <span v-else>{{ props.item.user.lastErrorMsg }}</span>
          </td>
        </template>
      </v-data-table>
    </v-card>
  </div>
</template>

<script lang="ts">
import Vue from "vue";
import Extension from "@/service/extension";
import {
  EAction,
  Site,
  LogItem,
  EModule,
  EUserDataRequestStatus,
  Options,
  UserInfo,
  EViewKey
} from "@/interface/common";
import dayjs from "dayjs";

const extension = new Extension();
export default Vue.extend({
  data() {
    return {
      words: {
        title: "概览（Beta）",
        getInfos: "刷新用户数据",
        cancelRequest: "取消请求",
        requesting: "正在请求",
        secret: "密",
        siteName: "网站名称",
        userName: "用户名称",
        userLevel: "用户等级",
        week: "时间显示为周数",
        timeline: "时间轴",
        settings: "参数",
        statistic: "数据图表",
        newMessage: "新消息"
      },
      loading: false,
      items: [] as any[],
      pagination: {
        rowsPerPage: -1
      },
      headers: [
        { text: "站点", align: "center", value: "name", width: "110px" },
        { text: "用户名", align: "left", value: "user.name" },
        { text: "等级", align: "left", value: "user.levelName" },
        {
          text: "数据量",
          align: "right",
          value: "user.uploaded",
          width: "120px"
        },
        { text: "分享率", align: "right", value: "user.ratio" },
        { text: "做种数", align: "right", value: "user.seeding" },
        { text: "做种体积", align: "right", value: "user.seedingSize" },
        { text: "魔力值/积分", align: "right", value: "user.bonus" },
        { text: "入站时间", align: "right", value: "user.joinTime" },
        { text: "数据更新于", align: "right", value: "user.lastUpdateTime" },
        { text: "状态", align: "center", value: "" }
      ],
      options: this.$store.state.options,
      beginTime: null as any,
      reloadCount: 0,
      requestQueue: [] as any[],
      requestTimer: 0,
      requestMsg: "",
      sites: [] as any[],
      filterKey: "",
      isSecret: false,
      showUserName: true,
      showSiteName: true,
      showUserLevel: true,
      showWeek: false
    };
  },
  created() {
    this.init();
  },

  methods: {
    resetSites() {
      this.sites = [];
      this.options.sites.forEach((site: Site) => {
        if (site.allowGetUserInfo) {
          if (!site.user) {
            site.user = {
              id: "",
              name: "",
              isLogged: false,
              isLoading: false
            };
          } else {
            this.formatUserInfo(site.user);
          }
          this.sites.push(site);
        }
      });
    },

    init() {
      extension
        .sendRequest(EAction.readConfig)
        .then((options: Options) => {
          this.options = this.clone(options);
          this.resetSites();
        })
        .catch();

      let viewOptions = this.$store.getters.viewsOptions(EViewKey.home, {
        showUserName: true,
        showSiteName: true,
        showUserLevel: true,
        showWeek: false
      });
      Object.assign(this, viewOptions);
    },
    getInfos() {
      this.loading = true;
      this.beginTime = dayjs();
      this.writeLog({
        event: `Home.getUserInfo.Start`,
        msg: `准备开始获取个人数据`
      });

      this.sites.forEach((site: Site, index: number) => {
        this.writeLog({
          event: `Home.getUserInfo.Processing`,
          msg: "正在获取 [" + site.name + "] 个人数据",
          data: {
            host: site.host,
            name: site.name
          }
        });

        this.getSiteUserInfo(site);
      });
    },
    /**
     * 记录日志
     * @param options
     */
    writeLog(options: LogItem) {
      extension.sendRequest(EAction.writeLog, null, {
        module: EModule.options,
        event: options.event,
        msg: options.msg,
        data: options.data
      });
    },

    /**
     * 移除搜索队列
     */
    removeQueue(site: Site) {
      let index = this.requestQueue.findIndex((item: any) => {
        return item.host === site.host;
      });
      (site.user as any).isLoading = false;
      if (index !== -1) {
        this.requestQueue.splice(index, 1);
        if (this.requestQueue.length == 0) {
          this.requestMsg = `请求完成，耗时：${dayjs().diff(
            this.beginTime,
            "second",
            true
          )} 秒。`;
          this.loading = false;
          this.writeLog({
            event: `Home.getUserInfo.Finished`,
            msg: this.requestMsg
          });
          // 重置站点信息，因为有时候加载完成后，某些行还显示正在加载，暂时未明是哪里问题
          this.sites = this.clone(this.sites);
        }
      }
    },
    /**
     * 格式化一些用户信息
     */
    formatUserInfo(user: UserInfo) {
      let downloaded = user.downloaded as number;
      let uploaded = user.uploaded as number;
      // 没有下载量时设置分享率为无限
      if (downloaded == 0 && uploaded > 0) {
        user.ratio = -1;
      }
      // 没有分享率时，重新以 上传量 / 下载量计算
      else if (downloaded > 0 && !user.ratio) {
        user.ratio = uploaded / downloaded;
      }
    },
    /**
     * 获取站点用户信息
     */
    getSiteUserInfo(site: Site) {
      if (!site.user) {
        return;
      }

      let user = site.user;
      user.isLoading = true;
      user.lastErrorMsg = "";

      this.requestQueue.push(Object.assign({}, site));
      extension
        .sendRequest(EAction.getUserInfo, null, site)
        .then((result: any) => {
          console.log(result);
          if (result && result.name) {
            user = Object.assign(user, result);
            this.formatUserInfo(user);
          }
        })
        .catch(result => {
          console.log("error", result);
          if (result.msg && result.msg.status) {
            user.lastErrorMsg = result.msg.msg;
          } else {
            user.lastErrorMsg = "发生错误";
          }
        })
        .finally(() => {
          this.removeQueue(site);
        });
    },

    abortRequest(site: Site) {
      extension
        .sendRequest(EAction.abortSearch, null, site)
        .then(() => {
          this.writeLog({
            event: `Home.getUserInfo.Abort`,
            msg: `${site.name} 获取用户资料请求已取消`
          });
        })
        .catch(() => {
          this.writeLog({
            event: `Home.getUserInfo.Abort.Error`,
            msg: `${site.name} 获取用户资料请求取消失败`
          });
          this.removeQueue(site);
        });
    },

    /**
     * 用JSON对象模拟对象克隆
     * @param source
     */
    clone(source: any) {
      return JSON.parse(JSON.stringify(source));
    },

    updateViewOptions() {
      this.$store.dispatch("updateViewOptions", {
        key: EViewKey.home,
        options: {
          showUserName: this.showUserName,
          showSiteName: this.showSiteName,
          showUserLevel: this.showUserLevel,
          showWeek: this.showWeek
        }
      });
    }
  },

  filters: {
    formatRatio(v: any) {
      if (v > 10000 || v == -1) {
        return "∞";
      }
      let number = parseFloat(v);
      if (isNaN(number)) {
        return "";
      }
      return number.toFixed(2);
    }
  }
});
</script>

<style lang="scss" >
.home {
  table.v-table thead tr:not(.v-datatable__progress) th,
  table.v-table tbody tr td {
    padding: 5px !important;
    font-size: 12px;
  }

  .center {
    text-align: center;
  }
  .number {
    text-align: right;
  }

  .nodecoration {
    text-decoration: none;
  }

  .messageCount {
    font-size: 9px;
    height: 16px;
    width: 16px;
    top: -5px;
    right: -16px;
  }
}
</style>
