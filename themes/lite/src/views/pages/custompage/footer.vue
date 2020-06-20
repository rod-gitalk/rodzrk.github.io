<template>
    <div>{{parentData.title}}</div>
</template>

<script>
    export default {
        name: "footer",
        props:{
          parentData:{}
        },
        data(){
            return{
              name:"张三"
            }
        },
        methods:{
          showPranetData(){
            console.log(">>>>>>>>",this.parentData);
          },
          showParData:function () {

          }
        },
        created() {
          this.showPranetData();
        }
    }
</script>

<style scoped>

</style>
