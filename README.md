# Vue-componentization-optimization
优化项目中的多个table在同一个页面的问题，封装成独立组建，动态渲染组件，抽取公共方法




项目中的一个sinkCom.vue需要切换多个table表格，发现代码全都写到一个面中长达上千行，通过用动态组件的方法讲每个table页面拆分到不同的vue文件中，再讲公用的编辑和删除功能提出出来，如果后续增加新的table会更加的方便
###### 使用componet动态渲染组件
```js
<template>
    <div class="secpage-con" style="overflow-y: auto" id="top-to-sinks-pane">
        <div class="mec-form-contain">
            <component
                :is="tbType"
                :tbType="tbType"
                :projectId="projectId"
                @toPage="toPage"
                :dataSource="dataSource"
                @deleteData="deleteData"
                @editData="editData"
            ></component>
        </div>
    </div>
</template>
```
这里的tbType会动态切换成kafka-sink、hbase-sink、jdbc-sink，通过组件名的切换来达到切换table的效果

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f925d3e9dbdd40608edec109e20f5ca6~tplv-k3u1fbpfcp-watermark.image?)

引入组件 
tbType为'hbase-sink'对应 HbaseSink
```js
import KafkaSink from "../table/sink/KafkaTable.vue";
import HbaseSink from "../table/sink/HBaseTable.vue";
import JdbcSink from "../table/sink/JdbcTable.vue";
```
注册组件

```js
components: {
    KafkaSink,
    JdbcSink,
    HbaseSink,
},
```

```js
//例子
<component
    //:is="kafka-sink"
    :is="tbType"
    ...
></component>
```
component会根据is的内容显示组件，我们用切换tbType来达到动态组件的效果


##### 独立数据查询方法
每个表格数据可能不同，所以获取表格的数据独立在每个table的vue文件中,但是数据统一命名，比如tableData，currentPage等后续可以抽离出方法在父组件中封装成统一的方法进行调用，方便很多
```js
data() {
    return {
        paging: {
            tableData: [],
            currentPage: 1,
            count: 0,
            pageSize: 10,
        },
    };
},
//获取表格数据
fetchData() {
    let vm = this;
    vm.loading = true;
    http.doRequest("yoururl", {
        currentPage: vm.paging.currentPage,
        pageSize: vm.paging.pageSize,
        projectId: vm.projectId,
        name: vm.kafkaName,
    }).then((result) => {
        vm.loading = false;
        let res = result.data;
        if (res.code == 0) {
            vm.paging.tableData = res.data
            //...
        }
   })
```
##### 组件的渲染和销毁

```js
mounted() {
    vebOn("getFetchData", (vedData) => {
        this.fetchData();
    });
    this.fetchData();
},
beforeDestroy() {
    // 销毁
    vebOff("getFetchData");
},
```
##### 抽离 编辑 和 删除 功能
每个表格都有编辑的方法，所以统一抽出来在父组件中进行调用，在函数中判断不同的子组件触发变化icon
```js
//sinkCom.vue
<component
    ...
    @editData="editData"
></component>

editData(data, type) {
    console.log('editData')
    this.toPage(data.name, type, data.id, data.foldId);
},
//根据不同组件触发同一个编辑函数
toPage(label, comPage, id) {
    let icon = {
        CsKafka: "icon-kafka",
        CsWebhook: "icon-webhook",
        CsHBase: "icon-hbase",
        CsKudu: "icon-kudu",
        CsJdbc: "icon-jdbc",
        SinkFlinkDDL: "icon-flinkddl",
    };
    let tab = {
        label,
        comPage,
        id,
        icon: icon[comPage],
    };
    this.$store.commit("ADD_TAB", tab);
},

//kafka.vue
<el-tooltip
    class="table-tooltip"
    :open-delay="1000"
    :content="'编辑'"
    placement="top"
>
    <div
        class="
            table-icon-con
            flex-vertical-center
        "
        @click.stop="
            $emit('editData',scope.row, 'CsKafka')
        "
    >
        <span
            class="table-icon icon-edit"
        ></span>
        <span>编辑</span>
    </div>
</el-tooltip>
```
##### 抽离页码和当前页变化方法
这里就是为什么之前我们要统一子组件中的tableData等方法的命名，在页码变化成功的时候直接调用组件的fetchData来重新获取数据，`SizeChange`和`CurrentChange`中做的调用和判断就会更加简洁，

```js
//kafka.vue
<div class="pane-pager">
    <el-pagination
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :current-page="paging.currentPage"
        :page-sizes="[10, 20, 30, 40]"
        :page-size="10"
        layout="total, sizes, prev, pager, next, jumper"
        :total="paging.count"
    ></el-pagination>
</div>
handleSizeChange(size) {
    this.$emit('SizeChange','kafka-sink',size)
},
handleCurrentChange(current) {
    console.log(123)
    this.$emit('CurrentChange','kafka-sink',current)
},
```

```js
//sinkCon.vue
<component
    ....
    @SizeChange="SizeChange"
    @CurrentChange="CurrentChange"
></component>
//这里的ref指的是组件的命名，在引入组件注册那一步声明的
SizeChange(ref,size){
    console.log('sizechange',ref,size)
    //this.$refs['kafka-sink'].paging.currentPage = 1;//变量具体
    this.$refs[ref].paging.currentPage = 1;
    this.$refs[ref].paging.pageSize = size;
    this.$refs[ref].fetchData()
},
CurrentChange(ref,size){
    console.log('currentchange',ref,size)
    this.$refs[ref].paging.currentPage = size;
    this.$refs[ref].fetchData()
},
```

![bbb.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d077477f39743faae48ced9b9fbb695~tplv-k3u1fbpfcp-watermark.image?)
