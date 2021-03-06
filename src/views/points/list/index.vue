<template>
  <div>
    <c-list
      :data="list.items"
      :columns="cList.columns"
      :total="list.total"
      :page-current="listPageCurrent"
      :search-where="listSearchWhere"
      :statistic="`总积分：${cList.total}`"
      @selection-change="handleListSelectionChange"
    >
      <c-list-header>
        <c-list-operations>
          <c-bulk-delete
            :selected-items="listSelectedItems"
            @ok="confirmDelete"
          >
          </c-bulk-delete>
          <Button type="primary" @click="$refs.form.show()">
            积分增减
          </Button>
        </c-list-operations>
        <c-list-search>
          <Form
            class="c-form c-form--search"
            inline
            @submit.native.prevent="search"
          >
            <Form-item prop="dateRange">
              <c-date-range
                class="c-form__input"
                :value="cList.cSearch.where.dateRange.$eq"
                @change="
                  date => {
                    cList.cSearch.where.dateRange.$eq = date;
                  }
                "
              ></c-date-range>
            </Form-item>
            <Form-item prop="wxUserId">
              <c-school-wx-user-select
                :school-id="getSchoolId()"
                :value="cList.cSearch.where.wxUserId.$eq"
                @change="
                  value => {
                    this.cList.cSearch.where.wxUserId.$eq = value;
                  }
                "
              >
              </c-school-wx-user-select>
            </Form-item>
            <Form-item>
              <Button type="primary" @click="search">
                查询
              </Button>
            </Form-item>
          </Form>
        </c-list-search>
      </c-list-header>
    </c-list>
    <c-list-form ref="form" @get-list="getListAndTotal"></c-list-form>
  </div>
</template>

<script>
import { Component, Vue } from "vue-property-decorator";
import { mapState } from "vuex";
import RouteParamsMixin from "view-ui-admin/src/mixins/route-params";
import ListMixin from "view-ui-admin/src/mixins/list";
import ListForm from "./form";

const module = "points";
const initWhere = {
  dateRange: {
    $eq: []
  },
  wxUserId: {
    $eq: ""
  }
};

@Component({
  components: {
    "c-list-form": ListForm
  },
  mixins: [RouteParamsMixin, ListMixin],
  computed: mapState({
    list: state => state[module].list
  })
})
export default class extends Vue {
  data() {
    const { ListColumnWidth } = this.$consts;

    return {
      cList: {
        columns: [
          {
            type: "selection",
            width: 60
          },
          {
            title: "微信用户",
            render: (h, { row }) => h("span", row.wxUser.nickName)
          },
          {
            title: "积分",
            key: "value",
            width: 100
          },
          {
            title: "时间",
            width: ListColumnWidth.CreatedAt,
            render: (h, { row }) => h("span", this.$time.getTime(row.createdAt))
          },
          {
            title: "操作",
            key: "action",
            width: 104,
            render: (h, { row }) =>
              h("div", [
                h("c-confirm-button", {
                  props: {
                    buttonText: "删除",
                    confirmText: "确认删除？"
                  },
                  on: {
                    ok: () => {
                      this.confirmDelete(row.id);
                    }
                  }
                })
              ])
          }
        ],
        cSearch: {
          where: this.$helpers.deepCopy(initWhere)
        },
        total: 0
      }
    };
  }

  async beforeRouteUpdate(to, from, next) {
    this.initListSearchWhere(initWhere);
    await this.getListAndTotal();
    next();
  }

  async created() {
    this.initListSearchWhere(initWhere);
    await this.getListAndTotal();
  }

  async getListAndTotal() {
    this.getList();
    this.cList.total = await this.getTotal();
  }

  getList() {
    return this.$store.dispatch(`${module}/getList`, {
      query: {
        offset: (this.listPageCurrent - 1) * this.$consts.PageSize,
        limit: this.$consts.PageSize,
        where: this.listSearchWhere,
        include: [
          {
            model: "WxUser",
            as: "wxUser"
          }
        ]
      }
    });
  }

  async getTotal() {
    const { data } = await this.$store.dispatch(`${module}/postAction`, {
      action: "getTotal",
      query: {
        where: this.listSearchWhere
      }
    });
    return data;
  }

  async confirmDelete(ids) {
    await this.$store.dispatch(`${module}/delete`, { id: ids });
    this.$Message.success("删除成功");

    const { items } = await this.getList();
    !items.length && this.goListPrevPage();
  }
}
</script>
