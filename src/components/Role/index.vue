<template>
  <div>
    <!-- 定义一个抽屉组件，标题是：分配角色权限，支持右上角关闭，禁用模态窗关闭，关闭抽屉触发closeDrawer方法，drawerFlag标识是否显示 -->
    <Drawer title="分配角色权限" :closable="true" :mask-closable="false"
        @on-close="closeDrawer" v-model="drawerFlag">
        <Button size="small" type="primary" @click="savePermission">保存</Button>
        <Button size="small" style="margin-left: 10px" @click="closeDrawer">关闭</Button>
        <Divider/>
        <!-- 定义tree组件，数据来源：permissionData，显示复选框，支持多选 -->
        <Tree :data="permissionData" show-checkbox multiple scrollable ref="permissionTree"></Tree>
        <Spin size="large" fix v-if="spinShow"></Spin>
      </Drawer>
  </div>
</template>
<script>

export default {
  name: 'RolePerminission',
  props: {},
  data() {
    return {
      // 当前分配的角色id
      roleId: '',
      // 标识抽屉组件是否打开
      drawerFlag: false,
      // 请求后端的时候页面显示加载中
      spinShow: true,
      // 所有模块
      allModuleList: [],
      // 当前分配角色已经拥有的权限
      multiPermissionList: [],
      // 当前登入用户拥有的权限
      userPermissionList: [],
      // tree的基础数据
      permissionData: [
        {
          title: '某某有限公司',
          expand: true,
          children: []
        }
      ]
    }
  },
  mounted() {},
  methods: {
    /**
     * 初始化数据，请求后端数据，
     * 这里要注意，请求一定要一个一个的请求，只有当前面的请求成功的时候才继续下面的请求
     * @param roleId 角色id
     */
    initData(roleId) {
      this.drawerFlag = true
      this.spinShow = true
      this.roleId = roleId
      // 获取所有的模块
      this.api({
        url:'',
        method:'get'
      }).then(data => {
    
        if (data.code === '00000000') {
          this.allModuleList = data.data.rows
          // 获取当前指定角色的权限
          this.api({
            url:'',
            method:'get'
          }).then(data => {
          
              if (data.code === '00000000') {
                this.multiPermissionList = data.data.rows
                // 获取当前登入用户的所有权限
                this.api({
                  url:'',
                  method:'get'
                }).then(data => {
                  
                    if (data.code === '00000000') {
                      this.userPermissionList = data.data.rows
                      // 当最后一个请求成功之后，调用初始化权限数据的方法
                      this.initPermissionData()
                    } else {
                      this.$Message.error(data.message)
                    }
                })
                
              } 
             
          })
         
        } 
      
      })
        
    },
    /**
     * 初始化权限数据
     */
    initPermissionData() {
      var moduleChildren = []
      // 对所有模块进行循环
      this.allModuleList.forEach(module => {
        var tempModuleChildren = {}
        // 定义模块名称（一级菜单），取后台请求到的模块名称
        tempModuleChildren.title = module.title
        // 默认展开
        tempModuleChildren.expand = true
        var permissionChildern = []
        // 循环当前登入用户的所有权限
        this.userPermissionList.forEach(permission => {
          if (permission.permissionModuleId === module.id) {
            // 如果moduleid相等，说明该权限是当前module下的权限
            var permissionId = permission.permissionId
            var tempPermissionChildern = {}
            // 定义权限名称（二级菜单），取后台请求到的权限名称
            tempPermissionChildern.title = permission.permissionTitle
            // 默认展开
            tempPermissionChildern.expand = true
            // 这里是关键，定义是否要选中，默认不选中
            var checked = false
            // 对当前需要分配的角色的权限进行循环
            this.multiPermissionList.forEach(multi => {
              if (multi.permissionId === permissionId) {
                // 如果权限id和当前循环的权限id相等，则说明角色有该权限，设置选中
                checked = true
              }
            })
            // 这里很重要，把权限id设置给当前二级菜单
            tempPermissionChildern.permissionId = permissionId
            tempPermissionChildern.checked = checked
            permissionChildern.push(tempPermissionChildern)
          }
        })
        tempModuleChildren.children = permissionChildern
        moduleChildren.push(tempModuleChildren)
      })
      this.permissionData[0].children = moduleChildren
      this.spinShow = false
    },
    // 关闭权限配置
    closeDrawer() {
      this.drawerFlag = false
      // 这里初始化tree的数据
      this.permissionData = [
        {
          title: '',
          expand: true,
          children: []
        }
      ]
    },
    /**
     * 保存权限
     */
    savePermission() {
      if (this.roleId === '') {
        this.$Message.error('请刷新页面重试')
        return
      }
      // 获取到当前tree的所有数据
      var checkData = this.$refs.permissionTree.data
      var permissionIds = []
      checkData.forEach(root => {
        // 对所有的权限数据进行遍历
        var moduleList = root.children
        moduleList.forEach(module => {
          var permissionList = module.children
          permissionList.forEach(permission => {
            if (permission.checked === true) {
              // 当前权限被选中，这里用到了我们先前定义好的权限id
              permissionIds.push(permission.permissionId)
            }
          })
        })
      })
      var params = []
      permissionIds.forEach(permission => {
        params.push({
          roleId: this.roleId,
          permissionId: permission
        })
      })
      // 调用保存接口，把当前权限保存到数据库
      batchSavePermission(params)
        .then(res => {
          const data = res.data
          if (data.code === 1001) {
            this.$Message.info("保存成功")
            // 保存成功关闭抽屉组件
            this.drawerFlag = false
          } else {
            this.$Message.error(data.message)
          }
        })
        .catch(err => {
          this.$Message.error(err)
        })
    }
  }
}
</script>
