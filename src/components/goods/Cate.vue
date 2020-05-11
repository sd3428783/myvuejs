<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>用户管理</el-breadcrumb-item>
      <el-breadcrumb-item>用户列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片视图 -->
    <el-card>
      <el-row>
        <el-col>
          <el-button type="primary" @click="showAddCateDialog">添加分类</el-button>
        </el-col>
      </el-row>
      <!-- 表格 -->
      <tree-table class="treeTable" :data="catelist" :columns="columns" :selection-type="false" :expand-type="false" show-index index-text="#" border :show-row-hover="false">
        <!-- //是否有效 -->
        <template slot="isok" slot-scope="scope">
          <i class="el-icon-success" v-if="scope.row.cat_deleted === false" style="color:lightgreen;"></i>
          <i class="el-icon-error" v-else style="color:red;"></i>
        </template>
        <!-- //排序 -->
        <template slot="order" slot-scope="scope">
          <el-tag size="mini" v-if="scope.row.cat_level === 0">一级</el-tag>
          <el-tag type="success" size="mini" v-else-if="scope.row.cat_level === 1">二级</el-tag>
          <el-tag type="warning" size="mini" v-else>三级</el-tag>
        </template>
        <!-- 操作 -->
        <template slot="opt" slot-scope="scope">
          <el-button type="primary" icon="el-icon-edit" size="mini" @click="showEditCateDialog(scope.row.cat_id)">编辑</el-button>
          <el-button type="danger" icon="el-icon-delete" size="mini"  @click="removeCateById(scope.row.cat_id)">删除</el-button>
        </template>
      </tree-table>
      <!-- 分页区域 -->
          <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="queryinfo.pagenum"
      :page-sizes="[3, 5, 10, 15]"
      :page-size="queryinfo.pagesize"
      layout="total, sizes, prev, pager, next, jumper"
      :total="total">
    </el-pagination>
    </el-card>
    <!-- 添加分类的对话框 -->
    <el-dialog
  title="添加分类"
  :visible.sync="addCateDialogVisble"
  width="50%" @close="addCateFormClosed">
  <!-- 添加分类的表单 -->
  <el-form :model="addCateForm" :rules="addCateFormRules" ref="addCateFormRef" label-width="100px">
  <el-form-item label="分类名称:" prop="cat_name">
    <el-input v-model="addCateForm.cat_name"></el-input>
  </el-form-item>
      <el-form-item label="父级分类：">
          <!-- options 用来指定数据源 -->
          <!-- props 用来指定配置对象 -->
          <el-cascader :options="parentCateList" :props="cascaderProps" v-model="selectedKeys" @change="parentCateChanged" clearable>
          </el-cascader>
        </el-form-item>
  </el-form>
  <span slot="footer" class="dialog-footer">
    <el-button @click="addCateDialogVisble = false">取 消</el-button>
    <el-button type="primary" @click="addCate">确 定</el-button>
  </span>
</el-dialog>
    <!-- 操作编辑的对话框 -->
    <el-dialog title="修改分类" :visible.sync="editCateDialogVisible" width="50%" @close="editDialogClosed">
      <el-form :model="cateEditForm" :rules="addCateFormRules" ref="editFormRef" label-width="80px">
        <el-form-item label="分类名称" prop="cat_name">
          <el-input v-model="cateEditForm.cat_name"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="editCateDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editRoleInfo">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      queryinfo: {
        type: 3,
        pagenum: 1,
        pagesize: 3
      },
      //商品分类得数据列表，默认为空
      catelist: [],
      //总数据条数
      total: 0,
      //为table指定列的定义
      columns:[{
          label:'分类名称',
          prop:'cat_name'
      },{
        label:'是否有效',
        //表示，当前列定义为模板列
        type:'template',
        //表示当前这一列使用的模板名称
        template:'isok'
      },{
        label:'排序',
        //表示，当前列定义为模板列
        type:'template',
        //表示当前这一列使用的模板名称
        template:'order'
      },{
        label:'操作',
        //表示，当前列定义为模板列
        type:'template',
        //表示当前这一列使用的模板名称
        template:'opt'
      }
      ],
      //控制添加分类对话框的显示与隐藏
      addCateDialogVisble:false,
      //添加分类的表单数据对象
      addCateForm:{
        //将要添加的分类名称
        cat_name:'',
        //父级分类的ID
        cat_pid:0,
        //分类的等级，默认要添加的是一级分类
        cat_level:0
      },
      addCateFormRules: {
        cat_name:[
          { required:true, message:'请输入分类名称',trigger:'blur'}
        ]
      },
      //父级分类的列表
      parentCateList:[],
      //指定级联选择器的配置对象
      cascaderProps:{
        expandTrigger:'hover',
        checkStrictly:false,
        value:'cat_id',
        label:'cat_name',
        children:'children',
      },
      //选中的父级分类的ID数组
      selectedKeys:[],
      editCateDialogVisible:false,
      cateEditForm:[]
    };
  },
  created() {
    this.getCateList();
  },
  methods: {
    // 获取商品分类数据
    async getCateList() {
      const { data: res } = await this.$http.get("categories", {
        params: this.queryinfo
      });

      if (res.meta.status !== 200) {
        return this.$message.error("获取商品分类失败！");
      }

      console.log(res.data);
      // 把数据列表，赋值给 catelist
      this.catelist = res.data.result;
      // 为总数据条数赋值
      this.total = res.data.total;
    },
    //监听pagesize改变
    handleSizeChange(newSize) {
      this.queryinfo.pagesize = newSize
      this.getCateList()
    },
    //监听pagenum改变
    handleCurrentChange(newPage) {
      this.queryinfo.pagenum = newPage
      this.getCateList()
    },
    showAddCateDialog() {
      this.getParentCateList()
      this.addCateDialogVisble = true
    },
    //获取父级分类的数据列表
    async getParentCateList() {
      const {data:res} = await this.$http.get('categories',{params:{type:2}})
      if(res.meta.status !== 200) {
        return this.$message.error("获取父级数据失败")
      }
      this.parentCateList = res.data
      console.log(res.data)
    },
    //选择项发生变化触发函数
    parentCateChanged() {
      console.log(this.selectedKeys)
      //如果selectKeys数组中的length大于0 ，证明选中了父级分类
      if(this.selectedKeys.length > 0) {
        this.addCateForm.cat_pid = this.selectedKeys[this.selectedKeys.length - 1]
        //为当前分类的等级赋值
        this.addCateForm.cat_level = this.selectedKeys.length
        return
      }else {
        //父级分类的ID
        this.addCateForm.cat_pid = 0
        //为当前分类的等级赋值
        this.addCateForm.cat_level = 0
      } 

    },
    //点击按钮，添加新的分类
    addCate() {
      // console.log(this.addCateForm)
      this.$refs.addCateFormRef.validate(async valid => {
        if(!valid) return
        const {data:res} = await this.$http.post('categories',this.addCateForm)
        if(res.meta.status !== 201) {
          return this.$message.error('添加分类失败')
        }
        this.$message.success('添加分类成功！')
        this.getCateList()
        this.addCateDialogVisble = false
      })
    },
    //监听对话框的关闭事件，重置表单数据
    addCateFormClosed() {
      this.$refs.addCateFormRef.resetFields()
      this.selectedKeys = []
      this.addCateForm.cat_pid = 0
      this.addCateForm.cat_level = 0
    },
    async showEditCateDialog(id) {
      const {data:res} = await this.$http.get('categories/' + id)
      console.log(res.data)
      if(res.meta.status !== 200) {
        return this.$message.error('分类信息加载失败')
      }
      this.cateEditForm = res.data
      this.editCateDialogVisible = true
    },
    editRoleInfo() {
      this.$refs.editFormRef.validate(async valid => {
        const {data:res} = await this.$http.put('categories/' + this.cateEditForm.cat_id,{
          cat_name:this.cateEditForm.cat_name
        })
        if(res.meta.status !== 200) {
          return this.$message.error('更新分类失败')
        }
        this.getCateList()
        this.$message.success('更新分类成功')
        this.editCateDialogVisible = false
      })
    },
    editDialogClosed() {
      this.$refs.editFormRef.resetFields()
    },
    async removeCateById(id) {
      const confirmResult = await this.$confirm(
        "此操作将永久删除该分类, 是否继续?",
        "提示",
      {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning"
      }
      ).catch(err => err);
      //如果用户确认删除，则返回值是字符串confirm
      //如果用户取消删除，则返回值为字符串cancel
      if(confirmResult !== 'confirm') {
        return this.$message.info('已取消了删除')
      }
      const {data:res} = await this.$http.delete('categories/' + id)
      if(res.meta.status !== 200) {
        return this.$message.error('删除失败')
      }
      this.getCateList()
      this.$message.success('删除成功')
    }
  }
};
</script>

<style>
.treeTable {
  margin-top: 15px;
}
.el-cascader {
  width: 100%;
}
</style>