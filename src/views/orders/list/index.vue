<template>
  <c-list
    :data="list.items"
    :columns="cList.columns"
    :total="list.total"
    :page-current="listPageCurrent"
    :search-where="listSearchWhere"
    @selection-change="handleListSelectionChange"
  >
    <c-list-header>
      <c-list-operations>
        <Button v-if="false" type="primary" @click="deliver">
          配送订单
        </Button>
        <c-bulk-delete :selected-items="listSelectedItems" @ok="confirmDelete">
        </c-bulk-delete>
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
          <Form-item prop="status">
            <Select
              class="c-form__input"
              placeholder="请选择状态"
              clearable
              v-model="cList.cSearch.where.status.$eq"
            >
              <Option
                v-for="item in dicts.OrderStatus"
                :key="item.value"
                :value="item.value"
                :label="item.label"
              >
                {{ item.label }}
              </Option>
            </Select>
          </Form-item>
          <Form-item prop="payment">
            <Select
              class="c-form__input"
              placeholder="请选择支付方式"
              clearable
              v-model="cList.cSearch.where.payment.$eq"
            >
              <Option
                v-for="item in dicts.OrderPayment"
                :key="item.value"
                :value="item.value"
                :label="item.label"
              >
                {{ item.label }}
              </Option>
            </Select>
          </Form-item>
          <Form-item prop="wxUserId">
            <c-wx-user-select
              :value="cList.cSearch.where.wxUserId.$eq"
              @change="
                value => {
                  this.cList.cSearch.where.wxUserId.$eq = value;
                }
              "
            >
            </c-wx-user-select>
          </Form-item>
          <Form-item v-if="false" prop="no">
            <Input
              class="c-form__input"
              placeholder="请输入订单号"
              v-model="cList.cSearch.where.no.$like"
            />
          </Form-item>
          <Form-item>
            <Button type="primary" @click="search">
              查询
            </Button>
          </Form-item>
          <Form-item>
            <Button type="primary" @click="exportXLSX">
              导出
            </Button>
          </Form-item>
        </Form>
      </c-list-search>
    </c-list-header>
  </c-list>
</template>

<script>
import { Component, Vue } from "vue-property-decorator";
import { mapState } from "vuex";
import RouteParamsMixin from "view-ui-admin/src/mixins/route-params";
import ListMixin from "view-ui-admin/src/mixins/list";
import xlsx from "view-ui-admin/src/utils/xlsx";
import Model from "@/models/admin/orders";

const module = "orders";
const initWhere = {
  dateRange: {
    $eq: []
  },
  payment: {
    $eq: ""
  },
  status: {
    $eq: ""
  },
  wxUserId: {
    $eq: ""
  }
};

@Component({
  mixins: [RouteParamsMixin, ListMixin],
  computed: mapState({
    list: state => state[module].list
  })
})
export default class extends Vue {
  data() {
    return {
      cList: {
        columns: [
          {
            type: "selection",
            width: 60
          },
          {
            title: "订单号",
            key: "no",
            width: 150
          },
          {
            title: "微信用户",
            key: "wxUser",
            width: 150,
            render: (h, { row }) => h("span", row.wxUser.nickName)
          },
          {
            title: "购买商品",
            key: "products",
            render: (h, { row }) => h("span", row.products[0].name)
          },
          {
            title: "支付方式",
            key: "payment",
            width: 90,
            render: (h, { row }) =>
              h(
                "span",
                this.$helpers.getItem(
                  this.dicts.OrderPayment,
                  "value",
                  row.payment
                )["label"]
              )
          },
          {
            title: "支付金额",
            key: "amount",
            width: 90,
            render: (h, { row }) => h("span", row.amount + " 元")
          },
          {
            title: "下单时间",
            key: "createdAt",
            width: 140,
            render: (h, { row }) => h("span", this.$time.getTime(row.createdAt))
          },
          {
            title: "状态",
            key: "status",
            width: 80,
            render: (h, { row }) =>
              h(
                "span",
                this.$helpers.getItem(
                  this.dicts.OrderStatus,
                  "value",
                  row.status
                )["label"]
              )
          },
          {
            title: "操作",
            key: "action",
            width: 192,
            render: (h, { row }) =>
              h("div", [
                h(
                  "Button",
                  {
                    props: {
                      disabled: !row.formData
                    },
                    on: {
                      click: () => {
                        this.cList.cFormData.data = row.formData;
                        this.cList.cFormData.fields =
                          row.products[0].formFields;
                        this.cList.cFormData.modal = true;
                      }
                    }
                  },
                  "表单信息"
                ),
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
        cFormData: {
          fields: {},
          data: {},
          modal: false
        }
      }
    };
  }

  async beforeRouteUpdate(to, from, next) {
    this.initListSearchWhere(initWhere);
    this.getList();
    next();
  }

  async created() {
    this.initListSearchWhere(initWhere);
    this.getList();
  }

  getList() {
    return this.$store.dispatch(`${module}/getList`, {
      query: {
        offset: (this.listPageCurrent - 1) * this.$consts.PageSize,
        limit: this.$consts.PageSize,
        where: this.listSearchWhere || initWhere,
        include: [{ model: "WxUser", as: "wxUser" }]
      }
    });
  }

  async confirmDelete(ids) {
    await this.$store.dispatch(`${module}/delete`, { id: ids });
    this.$Message.success("删除成功");

    const { items } = await this.getList();
    !items.length && this.goListPrevPage();
  }

  deliver() {}

  getFormData(data, fields) {
    return Object.keys(data).map(field => ({
      key: field,
      label: this.$helpers.getItem(fields, "key", field)["label"],
      value: data[field]
    }));
  }

  async getExportListItems() {
    this.search();

    const {
      data: { items }
    } = await new Model().GET({
      query: {
        where: this.listSearchWhere,
        include: [{ model: "WxUser", as: "wxUser" }]
      }
    });

    return items;
  }

  async exportXLSX() {
    const items = await this.getExportListItems();

    xlsx.download({
      fileName: `订单（${this.$time.getDate()}）`,
      data: (() => {
        return items.map(item => {
          let ret = {};

          console.log(item, "-");

          ret["订单号"] = item.no;
          ret["微信用户"] = item.wxUser.nickName;
          ret["购买商品"] = item.products[0].name;
          ret["支付方式"] = this.$helpers.getItem(
            this.dicts.OrderPayment,
            "value",
            item.payment
          )["label"];
          ret["支付金额"] = `${item.amount}元`;
          ret["下单时间"] = this.$time.getTime(item.createdAt);
          ret["状态"] = this.$helpers.getItem(
            this.dicts.OrderStatus,
            "value",
            item.status
          )["label"];

          return ret;
        });
      })()
    });
  }
}
</script>
