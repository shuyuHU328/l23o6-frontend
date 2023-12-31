<script setup lang="ts">
import {h, reactive, ref, watch} from 'vue'
import {SwitchFilled} from "@element-plus/icons-vue";
import {useStationsStore} from "~/stores/stations";
import {ElNotification} from "element-plus";
import {request} from "~/utils/request";
import {useRouter} from "vue-router";
import {RouteInfo} from '~/utils/interfaces';

const props = defineProps({
  name: String,
  route_id: Number,
  date: String,
  departure_times: Array,
  arrival_times: Array,
  extra_infos: Array,
  train_type: String,
  price_infos:Array
})


const router = useRouter()
const stations = useStationsStore()

let train = reactive({
  name: props.name,
  train_type: props.train_type,
  route_id: props.route_id,
  date: props.date,
  departure_times: props.departure_times as Array<string>,
  arrival_times: props.arrival_times as Array<string>,
  extra_infos: props.extra_infos as Array<string>,
  price_infos: props.price_infos as Array<Array<number>>
})
let route = reactive({
  id: 0,
  name: '',
  station_ids: []
})
let routes = ref([] as RouteInfo[])

const getRoutes = () => {
  request({
    url: `/admin/route`,
    method: 'GET'
  }).then((res) => {
    routes.value = res.data.data
  }).catch((error) => {
    if (error.response?.data.code == 100003) {
      router.push('/login')
    }
    ElNotification({
      offset: 70,
      title: 'getRoutes错误(trainManage)',
      message: h('error', {style: 'color: teal'}, error.response?.data.msg),
    })
    console.log(error)
  })
  console.log("end")
}

getRoutes()

const getRoute = () => {
  if (train.route_id === undefined) return
  request({
    url: `/admin/route/${train.route_id}`,
    method: 'GET'
  }).then((res) => {
    route.id = res.data.data.id
    route.name = res.data.data.name
    route.station_ids = res.data.data.station_ids
    if(train.price_infos.length<=1){
      train.price_infos = new Array(route.station_ids.length-1).fill(new Array(5).fill(0))
      for (let i = 0; i < train.price_infos.length; i++) {
        train.price_infos[i]=[200,150,100,50,20]
      }
    }
  }).catch((error) => {
    if (error.response?.data.code == 100003) {
      router.push('/login')
    }
    ElNotification({
      offset: 70,
      title: 'getRoute错误(trainManage)',
      message: h('error', {style: 'color: teal'}, error.response?.data.msg),
    })
    console.log(error)
  })
}

watch(() => train.route_id, () => {
  train.departure_times = []
  train.arrival_times = []
  train.extra_infos = []
  getRoute()
})
getRoute()
// const preType = [['商务座', '一等座', '二等座', '无座'], ['软卧', '硬卧', '软座', '硬座', '无座']]

</script>

<template>
  <div>
    <el-row>
      <el-col :span="7">
        <el-form-item>
          <template #label>
            <el-text tag="b" type="primary">
              车次名
            </el-text>
          </template>
          <el-input v-model="train.name"/>
        </el-form-item>
      </el-col>
      <el-col :span="7" :offset="1">
        <el-form-item style="display: flex">
          <template #label>
            <el-text tag="b" type="primary">
              车型
            </el-text>
          </template>
          <el-select v-model="train.train_type" style="display: flex; flex-grow: 1">
            <el-option v-for="type in ['高铁', '普通列车']" :key="type" :label="type" :value="type"/>
          </el-select>
        </el-form-item>
      </el-col>
      <el-col :span="8" :offset="1">
        <el-form-item style="display: flex">
          <template #label>
            <el-text tag="b" type="primary">
              日期
            </el-text>
          </template>
          <el-date-picker v-model="train.date" value-format="YYYY-MM-DD" :clearable="false"
                          style="display: flex; flex-grow: 1"/>
        </el-form-item>
      </el-col>
    </el-row>

    <el-row>
      <el-col :span="24">
        <el-form-item>
          <template #label>
            <el-text tag="b" type="primary">
              路线名
            </el-text>
          </template>
          <el-select v-model="train.route_id" style="width: 100%">
            <el-option v-for="singleRoute in routes" :key="singleRoute.id" :label="singleRoute.name"
                       :value="singleRoute.id"/>
          </el-select>
        </el-form-item>
      </el-col>
    </el-row>

    <div v-for="(station, index) in route.station_ids" :key="station">
      <el-card style="margin-bottom: 0.25%" shadow="hover" class="container">
        <div style="display: flex; align-items: center;">
          <el-icon class="handle" size="large">
            <SwitchFilled/>
          </el-icon>
          <strong style="margin-left: 5%; margin-right: 5%">
            {{ index + 1 }}
          </strong>
          <div style="width: 80%">
            {{ stations.idToName[station] }}
          </div>

          <el-date-picker style="width: 50%; margin-right: 1%" :disabled="index === 0"
                          @change="() => { if (index === route.station_ids.length - 1) { train.departure_times[index] = train.arrival_times[index] } }"
                          v-model="train.arrival_times[index]" type="datetime" placeholder="到点"
                          format="YY/MM/DD HH:mm"/>

          <el-date-picker style="width: 50%" :disabled="index === route.station_ids.length - 1"
                          @change="() => { if (index === 0) { train.arrival_times[0] = train.departure_times[0] } }"
                          v-model="train.departure_times[index]" type="datetime" placeholder="开点"
                          format="YY/MM/DD HH:mm"/>
        </div>
      </el-card>
    </div>
    <!--新增价格设置-->
    <el-text v-if="route.station_ids.length>0" tag="b" type="info">
      价格设置
    </el-text>
    <div v-for="(station, index) in route.station_ids" :key="station">
      <el-card v-if="index < route.station_ids.length -1 " style="margin-bottom: 0.25%" shadow="hover"
               class="container">
        <div style="display: flex; align-items: center;">
          <el-icon class="handle" size="large">
            <SwitchFilled/>
          </el-icon>
          <strong style="margin-left: 5%; margin-right: 5%">
            {{ index + 1 }}
          </strong>
          <div style="width: 80%">
            {{ stations.idToName[station] }} - {{ stations.idToName[route.station_ids[index + 1]] }}
          </div>
          <!--          {{train.price_infos[index][0]}}-->
          <div v-if="train.train_type=='高铁'">
            <el-row>
              <el-col :span="10">
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      商务座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][0]"/>
                </el-form-item>
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      一等座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][1]"/>
                </el-form-item>
              </el-col>
              <el-col :span="10">
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      二等座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][2]"/>
                </el-form-item>
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      无座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][3]"/>
                </el-form-item>
              </el-col>
            </el-row>
          </div>
          <div v-else>
            <el-row>
              <el-col :span="10">
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      软卧
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][0]"/>
                </el-form-item>
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      硬卧
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][1]"/>
                </el-form-item>
              </el-col>
              <el-col :span="10">
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      软座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][2]"/>
                </el-form-item>
                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      硬座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][3]"/>
                </el-form-item>

                <el-form-item style="margin-left: 5%">
                  <template #label>
                    <el-text tag="b">
                      无座
                    </el-text>
                  </template>
                  <el-input v-model="train.price_infos[index][4]"/>
                </el-form-item>
              </el-col>
            </el-row>
          </div>

        </div>
      </el-card>
    </div>

    <el-button @click="$emit('formSubmitted', train)" style="margin-top: 2vh" type="primary">
      确认
    </el-button>


    <!--        <pre>-->
    <!--          {{ train.arrival_times }}-->
    <!--          {{train.departure_times }}-->
    <!--        </pre>-->
  </div>
</template>

<style scoped>
.change {
  visibility: hidden
}

.container:hover .change {
  visibility: visible
}
</style>
