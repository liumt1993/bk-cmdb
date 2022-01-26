<template>
  <div
    class="group-wrapper"
    v-bkloading="{ isLoading: mainLoading }"
    :style="{ 'padding-top': topPadding + 'px' }"
  >
    <cmdb-main-inject
      ref="mainInject"
      inject-type="prepend"
      :class="['btn-group', 'clearfix', { sticky: !!scrollTop }]"
    >
      <cmdb-tips
        class="mb10"
        tips-key="modelTips"
        more-link="https://bk.tencent.com/docs/markdown/配置平台/产品白皮书/产品功能/Model.md"
      >
        {{ $t("模型顶部提示") }}
      </cmdb-tips>

      <div class="fl">
        <cmdb-auth :auth="{ type: $OPERATION.C_MODEL }">
          <bk-button
            slot-scope="{ disabled }"
            theme="primary"
            :disabled="disabled || modelType === 'disabled'"
            @click="showModelDialog()"
          >
            {{ $t("新建模型") }}
          </bk-button>
        </cmdb-auth>
        <cmdb-auth :auth="{ type: $OPERATION.C_MODEL_GROUP }">
          <bk-button
            slot-scope="{ disabled }"
            theme="default"
            :disabled="disabled || modelType === 'disabled'"
            @click="showGroupDialog(false)"
          >
            {{ $t("新建分组") }}
          </bk-button>
        </cmdb-auth>
        <!-- NOTE 需要添加导出权限 -->
        <cmdb-auth :auth="{ type: $OPERATION.C_MODEL_GROUP }">
          <bk-button
            slot-scope="{ disabled }"
            theme="default"
            :disabled="disabled || modelType === 'disabled'"
            @click="importModel"
          >
            {{ $t("导入") }}
          </bk-button>
        </cmdb-auth>
        <!-- NOTE 需要添加导入权限 -->
        <cmdb-auth :auth="{ type: $OPERATION.C_MODEL_GROUP }">
          <bk-button
            slot-scope="{ disabled }"
            theme="default"
            :disabled="disabled || modelType === 'disabled'"
            @click="exportModel"
          >
            {{ $t("导出") }}
          </bk-button>
        </cmdb-auth>
      </div>

      <div class="model-search-options fr">
        <bk-input
          class="search-model"
          :clearable="true"
          :right-icon="'bk-icon icon-search'"
          :placeholder="$t('请输入关键字')"
          v-model.trim="searchModel"
        >
        </bk-input>
      </div>

      <div class="model-type-options fr">
        <bk-button
          class="model-type-button"
          :class="[{ 'model-type-button-active': modelType === '' }]"
          size="small"
          @click="modelType = ''"
        >
          {{ $t("全部") }}
        </bk-button>
        <bk-button
          class="model-type-button"
          :class="[{ 'model-type-button-active': modelType === 'enable' }]"
          size="small"
          @click="modelType = 'enable'"
        >
          {{ $t("启用中") }}
        </bk-button>
        <span
          class="inline-block-middle"
          style="outline: 0"
          v-bk-tooltips="disabledModelBtnText"
        >
          <bk-button
            class="model-type-button disabled"
            :class="[{ 'model-type-button-active': modelType === 'disabled' }]"
            size="small"
            :disabled="!disabledClassifications.length"
            @click="modelType = 'disabled'"
          >
            {{ $t("已停用") }}
          </bk-button>
        </span>
      </div>
    </cmdb-main-inject>

    <ul class="group-list" v-show="currentClassifications.length">
      <li
        class="group-item clearfix"
        v-for="(classification, classIndex) in currentClassifications"
        :class="[{ 'is-collapse': classificationsCollapse[classification.id] }]"
        :key="classIndex"
      >
        <div class="group-title-wrapper clearfix" @click="toggleModelList(classification)">
          <div
            class="group-title"
            v-bk-tooltips="{
              disabled: isEditable(classification),
              content: $t('内置模型组不支持添加和修改'),
              placement: 'right'
            }"
          >
            <!-- eslint-disable-next-line max-len -->
            <bk-icon type="down-shape" />{{ classification["bk_classification_name"] }}&nbsp;(&nbsp;{{classification["bk_objects"].length}}&nbsp;)
          </div>
          <bk-dropdown-menu
            @click.native.stop
            v-if="isEditable(classification) && modelType !== 'disabled'"
            trigger="click"
            align="left">
            <template #dropdown-trigger>
              <div class="more-operation-btn">
                <bk-icon type="more" />
              </div>
            </template>
            <ul class="bk-dropdown-list" slot="dropdown-content">
              <li>
                <cmdb-auth
                  :auth="{
                    type: $OPERATION.C_MODEL,
                    relation: [classification.id]
                  }"
                >
                  <template #default="{ disabled }">
                    <dropdown-option-button
                      :disabled="disabled"
                      @click="showModelDialog(classification.bk_classification_id)"
                    >
                      {{$t('新建模型')}}
                    </dropdown-option-button>
                  </template>
                </cmdb-auth>
              </li>
              <li>
                <cmdb-auth
                  class="group-btn"
                  :auth="{
                    type: $OPERATION.U_MODEL_GROUP,
                    relation: [classification.id]
                  }"
                >
                  <template #default="{ disabled }">
                    <dropdown-option-button
                      :disabled="disabled"
                      @click="showGroupDialog(true, classification)"
                    >
                      {{$t('编辑分组')}}
                    </dropdown-option-button>
                  </template>
                </cmdb-auth>
              </li>
              <li>
                <cmdb-auth
                  class="group-btn"
                  :auth="{
                    type: $OPERATION.D_MODEL_GROUP,
                    relation: [classification.id]
                  }"
                >
                  <template #default="{ disabled }">
                    <dropdown-option-button
                      :disabled="disabled"
                      @click="deleteGroup(classification)"
                    >
                      {{$t('删除分组')}}
                    </dropdown-option-button>
                  </template>
                </cmdb-auth>
              </li>
            </ul>
          </bk-dropdown-menu>
        </div>
        <bk-transition name="collapse">
          <vue-draggable
            class="model-list clearfix"
            v-show="!classificationsCollapse[classification.id]"
            element="ul"
            draggable=".model-item"
            group="all-model-list"
            @change="handleDragChange">
            <li
              class="model-item bgc-white"
              :class="{
                'is-paused': model['bk_ispaused'],
                ispre: model.ispre
              }"
              v-for="(model, modelIndex) in classification['bk_objects']"
              :key="modelIndex"
            >
              <div
                class="info-model"
                :class="{
                  radius:
                    model.bk_ispaused ||
                    classification['bk_classification_id'] === 'bk_biz_topo'
                }"
                @click="modelClick(model)"
              >
                <div class="icon-box">
                  <i class="icon" :class="[model['bk_obj_icon']]"></i>
                </div>
                <div class="model-details">
                  <p class="model-name" :title="model['bk_obj_name']">
                    {{ model["bk_obj_name"] }}
                  </p>
                  <p class="model-id" :title="model['bk_obj_id']">
                    {{ model["bk_obj_id"] }}
                  </p>
                </div>
              </div>
              <div
                v-if="!model.bk_ispaused && model.bk_classification_id !== 'bk_biz_topo'"
                class="info-instance"
                @click="handleGoInstance(model)"
              >
                <p>{{ modelStatisticsSet[model.bk_obj_id] | instanceCount }}</p>
              </div>
            </li>
          </vue-draggable>
        </bk-transition>
      </li>
    </ul>
    <no-search-results
      v-if="!currentClassifications.length"
      :text="$t('搜不到相关模型')"
    />

    <bk-dialog
      class="bk-dialog-no-padding bk-dialog-no-tools group-dialog dialog"
      :close-icon="false"
      :width="600"
      :mask-close="false"
      v-model="groupDialog.isShow"
    >
      <div class="dialog-content">
        <p class="title">{{ groupDialog.title }}</p>
        <div class="content">
          <label>
            <div class="label-title">
              {{ $t("唯一标识") }}<span class="color-danger">*</span>
            </div>
            <div
              class="cmdb-form-item"
              :class="{ 'is-error': errors.has('classifyId') }"
            >
              <bk-input
                type="text"
                class="cmdb-form-input"
                name="classifyId"
                :placeholder="$t('请输入唯一标识')"
                :disabled="groupDialog.isEdit"
                v-model.trim="groupDialog.data['bk_classification_id']"
                v-validate="'required|classifyId|length:128'"
              >
              </bk-input>
              <p class="form-error" :title="errors.first('classifyId')">
                {{ errors.first("classifyId") }}
              </p>
            </div>
            <i
              class="bk-icon icon-info-circle"
              v-bk-tooltips="$t('请填写英文开头，下划线，数字，英文的组合')"
            ></i>
          </label>
          <label>
            <span class="label-title">
              {{ $t("名称") }}
            </span>
            <span class="color-danger">*</span>
            <div
              class="cmdb-form-item"
              :class="{ 'is-error': errors.has('classifyName') }"
            >
              <bk-input
                type="text"
                class="cmdb-form-input"
                name="classifyName"
                :placeholder="$t('请输入名称')"
                v-model.trim="groupDialog.data['bk_classification_name']"
                v-validate="'required|length:128'"
              >
              </bk-input>
              <p class="form-error" :title="errors.first('classifyName')">
                {{ errors.first("classifyName") }}
              </p>
            </div>
          </label>
        </div>
      </div>
      <div slot="footer" class="footer">
        <bk-button
          theme="primary"
          :loading="$loading(['updateClassification', 'createClassification'])"
          @click="saveGroup"
        >
          {{ groupDialog.isEdit ? $t("保存") : $t("提交") }}
        </bk-button>
        <bk-button theme="default" @click="hideGroupDialog">{{
          $t("取消")
        }}</bk-button>
      </div>
    </bk-dialog>

    <the-create-model
      :is-show.sync="modelDialog.isShow"
      :group-id.sync="modelDialog.groupId"
      :title="$t('新建模型')"
      @confirm="saveModel"
    >
    </the-create-model>

    <bk-dialog
      class="bk-dialog-no-padding"
      :width="400"
      :show-footer="false"
      :mask-close="false"
      v-model="sucessDialog.isShow"
    >
      <div class="success-content">
        <i class="bk-icon icon-check-1"></i>
        <p>{{ $t("模型创建成功") }}</p>
        <div class="btn-box">
          <bk-button theme="primary" @click="modelClick(curCreateModel)">{{
            $t("配置字段")
          }}</bk-button>
          <bk-button @click="sucessDialog.isShow = false">{{
            $t("返回列表")
          }}</bk-button>
        </div>
      </div>
    </bk-dialog>
  </div>
</template>

<script>
  import has from 'has'
  import cmdbMainInject from '@/components/layout/main-inject'
  import theCreateModel from '@/components/model-manage/_create-model'
  import noSearchResults from '@/views/status/no-search-results.vue'
  import dropdownOptionButton from '@/components/ui/button/dropdown-option-button.vue'
  import { mapGetters, mapMutations, mapActions } from 'vuex'
  import VueDraggable from 'vuedraggable'

  // NOTE 这两个事件监听似乎是无用的代码
  import {
    addMainScrollListener,
    removeMainScrollListener,
  } from '@/utils/main-scroller'
  // NOTE 这里的代码需要认真读一下
  import { addResizeListener, removeResizeListener } from '@/utils/resize-events'
  import {
    MENU_RESOURCE_HOST,
    MENU_RESOURCE_BUSINESS,
    MENU_RESOURCE_INSTANCE,
    MENU_MODEL_DETAILS,
  } from '@/dictionary/menu-symbol'

  export default {
    name: 'ModelManage',
    filters: {
      instanceCount(value) {
        if ([null, undefined].includes(value)) {
          return 0
        }
        return value > 999 ? '999+' : value
      },
    },
    components: {
      theCreateModel,
      cmdbMainInject,
      noSearchResults,
      dropdownOptionButton,
      VueDraggable
    },
    data() {
      return {
        scrollHandler: null,
        scrollTop: 0,
        topPadding: 0,
        modelType: '', // 模型启用/停用状态
        searchModel: '', // 模型搜索关键字
        filterClassifications: [], // 过滤后的分组
        modelStatisticsSet: {}, // 模型实例数量统计
        curCreateModel: {}, // 当前创建的模型
        // 创建成果弹窗
        sucessDialog: {
          isShow: false,
        },
        // 分组表单弹窗
        groupDialog: {
          isShow: false,
          isEdit: false,
          title: this.$t('新建分组'),
          data: {
            bk_classification_id: '',
            bk_classification_name: '',
            id: '',
          },
        },
        // 新建模型弹窗
        modelDialog: {
          isShow: false,
          groupId: '',
        },
        request: {
          statistics: Symbol('statistics'),
          searchClassifications: Symbol('searchClassifications'),
        },
        classificationsCollapse: {}
      }
    },
    computed: {
      // NOTE 如果管理不善，就会出现根本不知道从哪里引入的问题
      ...mapGetters(['supplierAccount', 'userName']),
      ...mapGetters('objectModelClassify', ['classifications']),
      allClassifications() {
        const allClassifications = []
        this.classifications.forEach((classification) => {
          allClassifications.push({
            ...classification,
            bk_objects: classification.bk_objects
              .filter(model => !model.bk_ishidden)
              .sort((a, b) => a.bk_ispaused - b.bk_ispaused),
          })
        })
        return allClassifications
      },
      enableClassifications() {
        const enableClassifications = []
        this.allClassifications.forEach((classification) => {
          enableClassifications.push({
            ...classification,
            bk_objects: classification.bk_objects.filter(model => !model.bk_ispaused),
          })
        })
        return enableClassifications.filter(item => item.bk_objects.length)
      },
      disabledClassifications() {
        const disabledClassifications = []
        this.classifications.forEach((classification) => {
          disabledClassifications.push({
            ...classification,
            bk_objects: classification.bk_objects.filter(model => model.bk_ispaused),
          })
        })
        return disabledClassifications.filter(item => item.bk_objects.length)
      },
      currentClassifications: {
        get() {
          if (!this.searchModel) {
            if (!this.modelType) return this.allClassifications
            return this.modelType === 'enable'
              ? this.enableClassifications
              : this.disabledClassifications
          }
          return this.filterClassifications
        },
        set(val) {
          console.log(val)
        }
      },
      disabledModelBtnText() {
        return this.disabledClassifications.length ? '' : this.$t('停用模型提示')
      },
      mainLoading() {
        return this.$loading(Object.values(this.request))
      },
    },
    watch: {
      currentClassifications: {
        deep: true,
        handler(val) {
          const classificationsCollapse = {}
          this.classificationsCollapse = val.forEach(({ id }) => {
            classificationsCollapse[id] = this.classificationsCollapse?.[id] || false
          })
          this.classificationsCollapse = classificationsCollapse
        }
      },
      searchModel(value) {
        if (!value) {
          return
        }
        const searchResult = []
        // eslint-disable-next-line no-nested-ternary
        const currentClassifications = !this.modelType
          ? this.allClassifications
          : this.modelType === 'enable'
            ? this.enableClassifications
            : this.disabledClassifications
        const classifications = this.$tools.clone(currentClassifications)
        const lowerCaseValue = value.toLowerCase()
        for (let i = 0; i < classifications.length; i++) {
          classifications[i].bk_objects = classifications[i].bk_objects.filter((model) => {
            const modelName = model.bk_obj_name.toLowerCase()
            const modelId = model.bk_obj_id.toLowerCase()
            // eslint-disable-next-line max-len
            return (
              (modelName && modelName.indexOf(lowerCaseValue) !== -1)
              || (modelId && modelId.indexOf(lowerCaseValue) !== -1)
            )
          })
          searchResult.push(classifications[i])
        }
        this.filterClassifications = searchResult.filter(item => item.bk_objects.length)
      },
      modelType() {
        this.searchModel = ''
      },
    },
    async created() {
      this.scrollHandler = (event) => {
        console.log('scroll handler')
        this.scrollTop = event.target.scrollTop
      }
      addMainScrollListener(this.scrollHandler)
      try {
        await Promise.all([
          this.getModelStatistics(),
          this.searchClassificationsObjects({
            params: {},
            config: {
              requestId: this.request.searchClassifications,
            },
          }),
        ])
      } catch (e) {
        this.$route.meta.view = 'error'
      }
      if (this.$route.query.searchModel) {
        const { hash } = window.location
        this.searchModel = this.$route.query.searchModel
        window.location.hash = hash.substring(0, hash.indexOf('?'))
      }
    },
    mounted() {
      addResizeListener(this.$refs.mainInject.$el, this.handleSetPadding)
    },
    beforeDestroy() {
      removeResizeListener(this.$refs.mainInject.$el, this.handleSetPadding)
      removeMainScrollListener(this.scrollHandler)
    },
    methods: {
      ...mapMutations('objectModelClassify', [
        'updateClassify',
        'deleteClassify',
      ]),
      ...mapActions('objectModelClassify', [
        'searchClassificationsObjects',
        'getClassificationsObjectStatistics',
        'createClassification',
        'updateClassification',
        'deleteClassification',
      ]),
      ...mapActions('objectModel', ['createObject']),
      toggleModelList(classification) {
        this.classificationsCollapse[classification.id] = !this.classificationsCollapse[classification.id]
      },
      handleDragChange() {},
      exportModel() {},
      importModel() {},
      handleSetPadding() {
        // NOTE 可以直接使用 CSS 来实现
        this.topPadding = this.$refs.mainInject.$el.offsetHeight
      },
      isEditable(classification) {
        return !['bk_biz_topo', 'bk_host_manage', 'bk_organization'].includes(classification.bk_classification_id)
      },
      showGroupDialog(isEdit, group) {
        if (isEdit) {
          this.groupDialog.data.id = group.id
          this.groupDialog.title = this.$t('编辑分组')
          this.groupDialog.data.bk_classification_id = group.bk_classification_id
          this.groupDialog.data.bk_classification_name =          group.bk_classification_name
          this.groupDialog.data.id = group.id
        } else {
          this.groupDialog.title = this.$t('新建分组')
          this.groupDialog.data.bk_classification_id = ''
          this.groupDialog.data.bk_classification_name = ''
          this.groupDialog.data.id = ''
        }
        this.groupDialog.isEdit = isEdit
        this.groupDialog.isShow = true
      },
      hideGroupDialog() {
        this.groupDialog.isShow = false
        this.$validator.reset()
      },
      async getModelStatistics() {
        const modelStatisticsSet = {}
        const res = await this.getClassificationsObjectStatistics({
          config: {
            requestId: this.request.statistics,
          },
        })
        res.forEach((item) => {
          modelStatisticsSet[item.bk_obj_id] = item.instance_count
        })
        this.modelStatisticsSet = modelStatisticsSet
      },
      async saveGroup() {
        const res = await Promise.all([
          this.$validator.validate('classifyId'),
          this.$validator.validate('classifyName'),
        ])
        if (res.includes(false)) {
          return
        }
        const params = {
          bk_supplier_account: this.supplierAccount,
          bk_classification_id: this.groupDialog.data.bk_classification_id,
          bk_classification_name: this.groupDialog.data.bk_classification_name,
        }
        if (this.groupDialog.isEdit) {
          // eslint-disable-next-line no-unused-vars
          const res = await this.updateClassification({
            id: this.groupDialog.data.id,
            params,
            config: {
              requestId: 'updateClassification',
            },
          })
          this.updateClassify({ ...params, ...{ id: this.groupDialog.data.id } })
        } else {
          const res = await this.createClassification({
            params,
            config: { requestId: 'createClassification' },
          })
          this.updateClassify({ ...params, ...{ id: res.id } })
        }
        this.hideGroupDialog()
        this.searchModel = ''
      },
      deleteGroup(group) {
        this.$bkInfo({
          title: this.$t('确认要删除此分组'),
          confirmFn: async () => {
            await this.deleteClassification({
              id: group.id,
            })
            this.$store.commit(
              'objectModelClassify/deleteClassify',
              group.bk_classification_id
            )
            this.searchModel = ''
          },
        })
      },
      showModelDialog(groupId) {
        this.modelDialog.groupId = groupId || ''
        this.modelDialog.isShow = true
      },
      async saveModel(data) {
        const params = {
          bk_supplier_account: this.supplierAccount,
          bk_obj_name: data.bk_obj_name,
          bk_obj_icon: data.bk_obj_icon,
          bk_classification_id: data.bk_classification_id,
          bk_obj_id: data.bk_obj_id,
          userName: this.userName,
        }
        const createModel = await this.createObject({
          params,
          config: { requestId: 'createModel' },
        })
        this.curCreateModel = createModel
        this.sucessDialog.isShow = true
        this.$http.cancel('post_searchClassificationsObjects')
        this.getModelStatistics()
        this.searchClassificationsObjects({
          params: {},
        })
        this.modelDialog.isShow = false
        this.modelDialog.groupId = ''
        this.searchModel = ''
      },
      modelClick(model) {
        this.$store.commit('objectModel/setActiveModel', model)
        this.$routerActions.redirect({
          name: MENU_MODEL_DETAILS,
          params: {
            modelId: model.bk_obj_id,
          },
          history: true,
        })
      },
      handleGoInstance(model) {
        this.sucessDialog.isShow = false
        const map = {
          host: MENU_RESOURCE_HOST,
          biz: MENU_RESOURCE_BUSINESS,
        }
        if (has(map, model.bk_obj_id)) {
          const query = model.bk_obj_id === 'host' ? { scope: 'all' } : {}
          this.$routerActions.redirect({
            name: map[model.bk_obj_id],
            query,
          })
        } else {
          this.$routerActions.redirect({
            name: MENU_RESOURCE_INSTANCE,
            params: {
              objId: model.bk_obj_id,
            },
          })
        }
      },
    },
  }
</script>

<style lang="scss" scoped>
.group-wrapper {
  background-color: #fafbfd;
}

.btn-group {
  position: absolute;
  top: 53px;
  left: 0;
  width: calc(100% - 17px);
  padding: 15px 20px 20px;
  font-size: 0;
  background-color: #fafbfd;
  z-index: 100;
  .bk-button {
    margin-right: 10px;
  }
  &.sticky {
    box-shadow: 0 0 8px 1px rgba(0, 0, 0, 0.03);
  }
}

.model-search-options {
  .search-model {
    width: 240px;
  }
}

.model-type-options {
  margin: 0 10px 0 0;
  font-size: 0;
  text-align: right;
  .model-type-button {
    position: relative;
    margin: 0;
    font-size: 12px;
    height: 32px;
    line-height: 30px;
    &.enable {
      border-radius: 2px 0 0 2px;
      border-right-color: #3a84ff;
      z-index: 2;
    }
    &.disabled {
      border-radius: 0 2px 2px 0;
      margin-left: -1px;
      z-index: 1;
    }
    &:hover {
      border-color: #3a84ff;
      z-index: 2;
    }
    &-active {
      border-color: #3a84ff;
      color: #3a84ff;
      z-index: 2;
    }
    & + .model-type-button {
      border-radius: 0 2px 2px 0;
      margin-left: -1px;
    }
  }
}

.group-list {
  padding: 0 24px;
}

.group-item {
  & + & {
    margin-top: 10px;
  }
}

.group-title-wrapper {
  display: inline-block;
  line-height: 26px;
  height: 26px;
  color: #313238;
  cursor: pointer;
  border-radius: 2px;
  &:hover {
    background-color: #f0f1f5;
  }
  .group-title {
    float: left;
    font-size: 14px;
    padding-right: 5px;
  }
  .bk-dropdown-menu{
    float: left;
  }
  .icon-down-shape {
    margin-left: 4px;
    margin-right: 4px;
    vertical-align: 1px;
    transition: transform 200ms ease;
    @at-root .group-item.is-collapse &{
      transform: rotate(180deg);
    }
  }
  .more-operation-btn {
    width: 26px;
    text-align: center;
    font-size: 14px;
    cursor: pointer;
    &:hover {
      background-color: #eaebf0;
      color: #3a84ff;
    }
  }
  .bk-dropdown-list {
    li,
    .auth-box {
      width: 78px;
    }
  }
}

.model-list {
  .model-item {
    float: left;
    display: flex;
    margin-top: 10px;
    margin-right: 12px;
    margin-bottom: 12px;
    width: 256px;
    // width: calc((100% - 10px * 4) / 5);
    height: 60px;
    background-color: #ffffff;
    border-radius: 2px;
    box-shadow: 0px 2px 4px 0px rgba(25,25,41,0.05);
    cursor: pointer;
    // &:nth-child(5n) {
    //   margin-right: 0;
    // }
    &.is-paused {
      opacity: 0.4;
    }
    &.ispre {
      .icon-box {
        color: #798aad;
      }
    }
    &:hover {
      border-color: $cmdbBorderFocusColor;
      .info-instance {
        display: block;
      }
    }
    .icon-box {
      flex: 0 0 40px;
      width: 40px;
      height: 40px;
      margin-top: auto;
      margin-bottom: auto;
      margin-left: 13px;
      line-height: 40px;
      text-align: center;
      border-radius: 50%;
      background-color: #e1ecff;
      .icon {
        color: #3a84ff;
        font-size: 16px;
        vertical-align: 1px;
      }
    }
    .model-details {
      margin-top: auto;
      margin-bottom: auto;
      margin-left: 11px;
      overflow: hidden;
    }
    .model-name {
      line-height: 19px;
      font-size: 14px;
      @include ellipsis;
    }
    .model-id {
      line-height: 16px;
      font-size: 12px;
      color: #bfc7d2;
      @include ellipsis;
    }
    .info-model {
      flex: 1 1 auto;
      display: flex;
      max-width: 100%;
      border-radius: 4px 0 0 4px;
      &:hover {
        background-color: #eff5ff;
      }
      &.radius {
        border-radius: 4px;
      }
    }
    .info-instance {
      display: none;
      width: 60px;
      line-height: 60px;
      text-align: center;
      margin-left: auto;
      border-radius: 0 4px 4px 0;
      color: #3a84ff;
      &:hover {
        background-color: #eff5ff;
      }
    }
  }
}

.dialog {
  .dialog-content {
    padding: 20px 15px 20px 28px;
  }
  .title {
    font-size: 20px;
    color: #333948;
    line-height: 1;
    padding-bottom: 14px;
  }
  .label-item,
  label {
    display: block;
    margin-bottom: 10px;
    font-size: 0;
    &:last-child {
      margin: 0;
    }
    .color-danger {
      display: inline-block;
      font-size: 16px;
      width: 15px;
      text-align: center;
      vertical-align: middle;
    }
    .icon-info-circle {
      font-size: 18px;
      color: $cmdbBorderColor;
    }
    .label-title {
      font-size: 16px;
      line-height: 36px;
      vertical-align: middle;
      @include ellipsis;
    }
    .cmdb-form-item {
      display: inline-block;
      margin-right: 10px;
      width: 519px;
      vertical-align: middle;
    }
  }
  .footer {
    font-size: 0;
    text-align: right;
    .bk-primary {
      margin-right: 10px;
    }
  }
}

.success-content {
  text-align: center;
  padding-bottom: 46px;
  p {
    color: #444444;
    font-size: 24px;
    padding: 10px 0 20px;
  }
  .icon-check-1 {
    width: 58px;
    height: 58px;
    line-height: 58px;
    font-size: 50px;
    font-weight: bold;
    color: #fff;
    border-radius: 50%;
    background-color: #2dcb56;
    text-align: center;
  }
  .btn-box {
    font-size: 0;
    .bk-button {
      margin: 0 5px;
    }
  }
}
</style>
