<template>
  <div id="root">
    <div class="header">
      <Row>
        <Col span="21">
        <Form ref="formInline" :model="form" :label-width="80" inline>
          <Form-item :rules="rules" prop="ip" label="IP地址：">
            <Input v-model="form.ip" placeholder="请输入IP地址"></Input>
          </Form-item>
          <Form-item prop="name" label="操作名称：">
            <Input v-model="form.name" placeholder="请输入操作名称"></Input>
          </Form-item>
          <Form-item prop="date" label="起止时间：">
            <Date-picker type="datetimerange" v-model="form.date" :options="dateOption" format="yyyy/MM/dd HH:mm" placeholder="选择日期和时间" style="width: 300px"></Date-picker>
          </Form-item>
          <div class="button">
            <Button type="primary" icon="ios-search" @click="handleSubmit()">搜索</Button>
            <Button type="ghost" @click="handleReset()" style="margin-left:8px">重置</Button>
          </div>
        </Form>
        </Col>
        <Col span="3">
        <Button class="export" type="primary" size="large" @click="exportData()">
          <Icon type="ios-download-outline" size="20"></Icon>导出
        </Button>
        </Col>
      </Row>
    </div>
    <Table ref="table" :data="data" :columns="columns" size="small" border highlight-row stripe>
    </Table>
    <div class="footer">
      <Page :total="total" :current="current" @on-change="changePage" @on-page-size-change="changeSize" size="small" placement="top" show-total show-sizer show-elevator></Page>
    </div>
  </div>
</template>
<script>
import { fetchOperateLog, downloadOperateLog } from '@/api/log/log';
import validate from '@/utils/validate';
import filters from '@/filters';
import { getStyle, debounce } from '@/utils/util';

export default {
  name: 'OperateLog',
  data() {
    return {
      form: {
        ip: '',
        name: '',
        date: []
      },
      dateOption: {
        disabledDate(date) {
          return date && date.valueOf() > Date.now();
        },
        shortcuts: [
          {
            text: '今天',
            value() {
              return [new Date().setHours(0, 0, 0, 0), Date.now()];
            }
          },
          {
            text: '昨天',
            value() {
              return [Date.now() - (3600 * 1000 * 24), Date.now()];
            }
          },
          {
            text: '一周前',
            value() {
              return [Date.now() - (3600 * 1000 * 24 * 7), Date.now()];
            }
          }
        ]
      },
      data: [],
      columns: [{
        title: '操作名称',
        key: 'operationname'
      },
      {
        title: '日志类型',
        key: 'logtypename'
      },
      {
        title: 'IP地址',
        key: 'userip'
      },
      {
        title: '操作者名称',
        key: 'username'
      },
      {
        title: '执行状态',
        key: 'logstatusname',
        align: 'center',
        render: (h, params) => {
          const row = params.row;
          const color = row.logstatus === 200 ? 'green' : 'red';
          return <tag type="dot" color={color}>{params.row.logstatusname}</tag>;
        },
        filterMultiple: false,
        filters: [
          {
            label: '成功',
            value: '成功'
          },
          {
            label: '失败',
            value: '失败'
          }
        ],
        filterMethod(value, row) {
          return row.logstatusname.includes(value);
        }
      },
      {
        title: '创建时间',
        key: 'logdate',
        align: 'center',
        render: (h, params) => <div>{filters.formatDate(params.row.logdate)}</div>
      },
      {
        title: '日志来源',
        key: 'operationsourcename',
        filters: [
          {
            label: '统计分析',
            value: '统计分析'
          },
          {
            label: '平台监控',
            value: '平台监控'
          },
          {
            label: '日志管理',
            value: '日志管理'
          }
        ],
        filterMethod(value, row) {
          return row.operationsourcename.includes(value);
        }
      }],
      rules: {
        type: 'string',
        pattern: validate.ip,
        message: '请输入正确的ip地址',
        trigger: 'blur'
      },
      tableHeight: 0,
      total: 0,
      current: 1
    };
  },
  computed: {
    filter() {
      return {
        ip: this.form.ip,
        name: this.form.name,
        start: +new Date(this.form.date[0]) || '',
        end: +new Date(this.form.date[1]) || ''
      };
    }
  },
  methods: {
    exportData() {
      if (this.total > 1000) {
        const fileds = this.columns.map(val => ({
          name: val.key,
          alias: val.title
        }));
        downloadOperateLog(fileds, this.filter);
      } else {
        this.$refs.table.exportCsv({
          filename: '操作日志',
          original: false,
          columns: this.columns,
          data: this.data.map(val => {
            val.logdate = filters.formatDate(val.logdate);
            return val;
          })
        });
      }
    },
    handleSubmit() {
      fetchOperateLog(this.filter, this.current).then(response => {
        this.data = response.data.dataSource;
        this.total = response.data.pageInfo.totalCount;
        this.$Message.success('刷新成功！');
      });
    },
    handleReset() {
      this.$refs.formInline.resetFields();
    },
    changePage(pageIndex) {
      this.current = pageIndex;
      fetchOperateLog(this.filter, this.current).then(response => {
        this.data = response.data.dataSource;
      });
    },
    changeSize(pageSize) {
      fetchOperateLog(this.filter, this.current, pageSize).then(response => {
        this.data = response.data.dataSource;
      });
    },
    calcTableHeight() {
      const totalHeight = getStyle(this.$el, 'height');
      const headerHeight = getStyle(document.getElementsByClassName('header')[0], 'height');
      const footerHeight = getStyle(document.getElementsByClassName('footer')[0], 'height');
      this.tableHeight = totalHeight - headerHeight - footerHeight;
    }
  },
  created() {
    this.handleSubmit();
  },
  mounted() {
    this.calcTableHeight();
    window.addEventListener('resize', debounce(this.calcTableHeight), false);
  },
  beforeDestroy() {
    window.removeEventListener('resize', debounce(this.calcTableHeight), false);
  }
};
</script>
<style lang="less" scoped>
#root {
  height: 100%;
  width: 100%;
}

.button {
  display: inline-block;
  margin-bottom: 24px;
  margin-right: 10px;
}

.export {
  .ivu-icon {
    margin-right: 10px;
  }
}

.footer {
  height: 45px;
  padding: 20px 0;
}
</style>