<!-- 
此组件初始化时不会自动加载数据，请在mounted中 this.$refs.[].initData()
如果放在dialog里面的吗，请在dialog的open事件中用this.$nextTick进行加载
 -->
<template>
	<div>
		<!-- 查询条件 -->
		<div style='margin-bottom: 2px' v-show='!hideQuery'>
			<div style='float:right;'>
				<el-button-group>
				  <el-button type='primary' icon="el-icon-search" @click='query'></el-button>
				  <el-tooltip content='重置查询条件' placement='top'>
					  <el-button icon="el-icon-refresh" @click='resetQuery'></el-button>
					</el-tooltip>
				  <!-- <el-tooltip content='导出Excel' placement='top'>
				  	<el-button @click='exportExcel' size='mini' icon='el-icon-download'></el-button>
					</el-tooltip> -->
				  <el-tooltip content='显示更多查询条件' placement='top'>
					  <el-button @click='queryShowMore=!queryShowMore' size='mini'>
	          <i :class="{'el-icon-arrow-up':queryShowMore,'el-icon-arrow-down':!queryShowMore}"></i>
          	</el-button>
          </el-tooltip>
				</el-button-group>
			</div>
			<el-form ref='formQuery' :model='queryParams' class='c-form-condensed' label-width='68px' inline size='mini'>
				<el-form-item label='员工姓名' prop='name'>
					<el-input v-model.trim='queryParams.name' clearable></el-input>
				</el-form-item>
				<el-form-item label='工号' prop='no'>
					<el-input v-model.trim='queryParams.no' clearable></el-input>
				</el-form-item>
				<div v-show='queryShowMore'>
					<el-form-item label='部门' prop='dep_id'>
						<el-input v-model='queryParamsLabel.dep_name' placeholder='点击选择' readonly clearable @click.native='openSelectDepDialog'>
							<i 
								style='cursor: pointer;'
								v-show='queryParams.dep_id' 
								slot="suffix" 
								class="el-input__icon el-icon-close" 
								@click.stop='queryParamsLabel.dep_name="";queryParams.dep_id=""'></i>
						</el-input>
					</el-form-item>
					<el-form-item label='子部门' prop='hasSubDep'>
						<el-switch
						  v-model="queryParams.hasSubDep"
						  :inactive-value="0"
						  :active-value="1">
						</el-switch>
					</el-form-item>
					<el-form-item label='离职' prop='is_disabled'>
						<el-switch
						  v-model="queryParams.is_disabled"
						  :inactive-value="0"
						  :active-value="1">
						</el-switch>
					</el-form-item>
				</div>
			</el-form>
		</div> 
		<!--/ 查询条件 -->		
		<!-- 数据表格 -->
		<el-table 
			:data='list' 
			ref='tableList'
			v-loading='loading'
			highlight-current-row
			border 
			stripe
			row-key='id'
			:size='size'
			:max-height='tableMaxHeight' 			
			@selection-change='selectionChange'
			:summary-method='getSummaryData'
			@sort-change='sortChange'>			
			<el-table-column 
				fixed
				v-if='showCheckbox'
				type='selection' 
				align='center' 
				width='35' />
			<el-table-column prop='name' label='姓名' min-width='100' show-overflow-tooltip/>
			<el-table-column prop='no' label='工号' width='100' show-overflow-tooltip/>
			<el-table-column prop='mail' label='邮箱' width='150' show-overflow-tooltip/>
			<el-table-column prop='dep_name' label='部门' width='100' show-overflow-tooltip/>
			<el-table-column prop='sex' label='性别' align='center' width='60'>
				<template slot-scope='{row}'>
					<el-tag v-if='row.sex==0'>男</el-tag>
					<el-tag v-else type='danger'>女</el-tag>
				</template>
			</el-table-column>
			<el-table-column 
				prop='create_user_name' 
				width='90' 
				label='录入员' />
			<el-table-column 
				prop='create_time' 
				width='120' 
				label='创建时间' 
				sortable='custom'>
				<template slot-scope='{ row }'>
					<span>{{ row.create_time | formatDate }}</span>
				</template>
			</el-table-column>
			<el-table-column 
				prop='update_time' 
				label='最近更新时间' 
				width='120' 
				sortable='custom'>
				<template slot-scope='{row}'>
					<span>{{ row.update_time | formatDate }}</span>
				</template>
			</el-table-column>
			<!-- slot[column] -->
			<slot name='column'></slot>
			<!--/ slot[column]-->
		</el-table>
		<!-- 数据表格 -->
		<!-- 分页 -->
		<el-pagination style='margin-top: 10px'
			v-if='!noPage'
	    :page-sizes = "[10, 20, 50, 100]"
	    :page-size= "requestParams.pageSize"
	    :current-page.sync = "requestParams.currentPage"
	    layout="total, sizes, prev, pager, next, jumper"
	    :total="dataTotal"
	    @size-change='sizeChange'
	    @current-change='getData' />
	  <!--/ 分页 -->
	  <asset-details :in-dialog='inDialog' ref='assetDetails' />
	  <dep-dialog :in-dialog='inDialog' ref='depDialog' show-select @select='selectDep' />
	</div>
</template>
<script>
import employeeApi from '@/api/hr/employee'
import assetDetails from '@/components/it/asset/details'
import depDialog from '@/components/hr/dep/treeDialog'

const initQueryParamsLabel = {
	dep_name:''
}
export default {
	components:{ assetDetails, depDialog },
	props:{
		size:{
			type:String,
			default:''
		},
		maxHeight:{
			default:0
		},
		params:{
			default:()=>({})
		},
		inDialog:{
			type:Boolean,
			default:false
		},
		hideQuery:{
			type:Boolean,
			default:false
		},
		noPage:{
			type:Boolean,
			default:false
		},
		init:{
			type:Boolean,
			default:false
		},
		showMore:{
			type:Boolean,
			default:false
		},
		showCheckbox:{
			type:Boolean,
			default:false
		}
	},
	data(){
		return {
			inited:false,
			loading: false,
			dialogShow:false,
			list:[],
			dataTotal:0,
			formLoading:false,
			projectList:[],
			summaryData:{},
			selectionList:[],
			queryShowMore:this.showMore,
			initParams:{},
			queryParamsLabel:{
				dep_name:''
			},
			//查询条件字段
			queryParams:{
				no:'',//项目编号				
				name:'',
				dep_id:'',
				hasSubDep:1
			},
			//数据请求的参数
			requestParams:{
				pageSize:this.$store.state.sys.pageSize,//分页大小
				currentPage:1,//当前页
				sortProp:'',
				sortOrder:'',
				noPage:this.noPage?1:0
			}
		}
	},
	computed:{
		tableMaxHeight(){
			return this.maxHeight?this.maxHeight:this.$store.state.sys.tableMaxHeight
		}
	},
	created(){
		
	},
	mounted(){
    if(this.init){
      this.inited = true
      this.initData()
    }    
  },
	methods:{
		//初始化数据
		initData(params={}){
			this.initParams = {...params}
			this.resetQuery()
		},
		//刷新数据
		reload(){
			this.getData()
		},
		//分页容量大小发生变化时
		sizeChange(value){
			this.requestParams.pageSize=value
			this.getData()
		},
		selectionChange(valueArrary){
			this.selectionList = valueArrary
		},
		getSummaryData({columns,data}){
      let sum = []
      let labelIndex = this.showCheckbox?1:0
      columns.forEach((column,i)=>{
        if(i==labelIndex){
          sum[i]='合计'
        }else{
          sum[i] = this.summaryData[column.property]
        }       
      })
      return sum
    },
		//获取数据
		getData() {
			this.loading=true
			employeeApi.getList({...this.requestParams,...this.params,...this.initParams}).then(res=>{
				this.list = res.data.list
				this.dataTotal = res.data.total
				this.summaryData = res.data.summary
				this.loading = false
				this.$emit('loaded',{ 
					total:this.dataTotal,
					list:this.list,
					params:{...this.queryParams} 
				})
			})
		},
		//查询方法
		query({ key }={}){
			if(key){
				let value = this.queryParams[key]
				this.requestParams = {...this.requestParams,[key]:value}
			}else{
				this.requestParams = {...this.requestParams,...this.queryParams}
			}			
			this.getData()
		},
		//重置查询条件
		resetQuery(){
			this.$refs.formQuery.resetFields()
			this.queryParamsLabel = { ...initQueryParamsLabel }
			// this.queryParams = {...this.queryParams,...this.params}
			this.requestParams.currentPage = 1
			this.query()
		},
		openDetails(row){
			this.$refs.assetDetails.open().then(that=>{
				that.initData(row)
			})
		},
		sortChange({prop,order}){
			this.requestParams.sortProp = prop
			this.requestParams.sortOrder = order
			this.getData()
		},
		//导出excel
		exportExcel(){
			employeeApi.exportExcel(this.requestParams)
		},
		clear(){
			this.list = []
		},
		del(row){
			let confirmText = '确定删除此员工吗？'
			this.$confirm(confirmText,'提示',{
				type: 'warning'
			}).then(()=>{
				employeeApi.del(row.id).then(res=>{
					this.reload()
					this.$message.success('删除成功')
					this.$emit('del')
				})
			})
		},
		openUseStatusDialog(row){
			this.$refs.assetUseStatusDialog.open().then(that=>{
				that.initData({asset_id:row.id})
			})
		},
		openSelectDepDialog(){
			this.$refs.depDialog.open().then(that=>{
				that.initData()
			})
		},
		selectDep(data){
			this.queryParamsLabel.dep_name = data.name
			this.queryParams.dep_id = data.id
			this.$refs.depDialog.close()
		}
	}
}
</script>