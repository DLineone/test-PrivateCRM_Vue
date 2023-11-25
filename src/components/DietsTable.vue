<script setup lang="ts">
  import TruckSvg from '@/components/images/TruckSvg.vue';
  import MessageSvg from '@/components/images/MessageSvg.vue';
  import CaretUp from '@/components/images/CaretUpSvg.vue';
  import CaretDown from '@/components/images/CaretDownSvg.vue';
  import plural from 'plural-ru';
  import type clientsType from "@/data.json";
  import { ref, computed, watch} from "vue";
  const props = defineProps<{clients: typeof clientsType, filter: number | ''}>();

  const sortingVariants = ['no', 'up', 'down', 'endtoday', 'endtomorrow'];
  let sortingState = ref(parseInt((new URLSearchParams(window.location.search)).get('sortingState') ?? '0'));
  watch(sortingState, () => {
    const url = new URL(window.location.toString());
    url.searchParams.set('sortingState', sortingState.value.toString());
    window.history.pushState(null, '', url.toString());
  })

  let statusClients = props.clients.map(client => {
    let startDates = client.dates.map(dates => Date.parse(dates.start_date));
    let endDates = client.dates.map(dates => Date.parse(dates.end_date));
    let today = Date.now();

    let daysToEnd = [];
    let daysToStart = [];
    for(let i = 0; i < startDates.length; i++)
    {
      let diffToEnd = endDates[i] - today;
      daysToEnd[i] = Math.ceil(diffToEnd / (1000 * 60 * 60 * 24));

      let diffToStart = startDates[i] - today;
      daysToStart[i] = Math.ceil(diffToStart / (1000 * 60 * 60 * 24));
    }
    return {...client, endsAt: daysToEnd, startsAt: daysToStart}
  });

  let tableClients = computed(() => {
    let cloneClients;
    if(props.filter !== '')
    {
      let numFilter: number = props.filter;
      cloneClients = structuredClone(statusClients).filter((client) => client.startsAt.includes(numFilter));
      return cloneClients;
    }
    else
    {
      cloneClients = structuredClone(statusClients);
    } 

    switch(sortingVariants[sortingState.value]){
      case 'no':
        return statusClients;
      case 'up':
        return cloneClients.sort((client1, client2) => {
          let start1max = Math.max(...client1.startsAt);
          let start2max = Math.max(...client2.startsAt);
          let end1max = Math.max(...client1.endsAt);
          let end2max = Math.max(...client2.endsAt);
          let end1min = Math.min(...client1.endsAt);
          let end2min = Math.min(...client2.endsAt);
          
          // Начинается через Х дней: между всеми и между собой
          if(start1max >= 0 || start2max >= 0)
          {
            if(start1max > start2max)
              return -1;
            if(start1max < start2max)
              return 1;
          }
          // Заканчивается через Х дней и Закончилось Х дней назад: между всеми
          if((start1max < 0 && start2max < 0) && (end1max >= 0 || end2max >= 0))
          {
            // Заканчивается через Х дней: между собой
            if(end1max >= 0 && end2max >=0)
            {
              if(end1max < end2max)
                return -1;
              if(end1max > end2max)
                return 1;
            }

            if(end1max > end2max)
              return -1;
            if(end1max < end2max)
              return 1;
          }
          // Закончилось Х дней назад: между собой
          if(end1min < 0 && end2min < 0)
          {
            if(end1min > end2min)
              return -1;
            if(end1min < end2min)
              return 1;
          }
      
          return 0;
        });
      case 'down':
        return cloneClients.sort((client1, client2) => {
          let start1max = Math.max(...client1.startsAt);
          let start2max = Math.max(...client2.startsAt);
          let end1max = Math.max(...client1.endsAt);
          let end2max = Math.max(...client2.endsAt);
          let end1min = Math.min(...client1.endsAt);
          let end2min = Math.min(...client2.endsAt);
          
          // Начинается через Х дней: между всеми и между собой
          if(start1max >= 0 || start2max >= 0)
          {
            if(start1max < start2max)
              return -1;
            if(start1max > start2max)
              return 1;
          }
          // Заканчивается через Х дней и Закончилось Х дней назад: между всеми
          if((start1max < 0 && start2max < 0) && (end1max >= 0 || end2max >= 0))
          {
            // Заканчивается через Х дней: между собой
            if(end1max >= 0 && end2max >=0)
            {
              if(end1max > end2max)
                return -1;
              if(end1max < end2max)
                return 1;
            }

            if(end1max < end2max)
              return -1;
            if(end1max > end2max)
              return 1;
          }
          // Закончилось Х дней назад: между собой
          if(end1min < 0 && end2min < 0)
          {
            if(end1min < end2min)
              return -1;
            if(end1min > end2min)
              return 1;
          }
      
          return 0;
        });
      case 'endtoday':
        return cloneClients.filter((client) => client.endsAt.includes(0));
      case 'endtomorrow':
        return cloneClients.filter((client) => client.endsAt.includes(1));
      default:
        return statusClients;
    }
  });

  function dateIntervalFromString(datestring: {start_date: string, end_date: string}, tariff: string)
  {
    let startDate = Date.parse(datestring.start_date);
    let endDate = Date.parse(datestring.end_date);
    
    let diffTime = Math.abs(endDate - startDate);
    let diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    
    let startMonth = (new Date(startDate)).toLocaleString('default', { month: 'short' }).match(/[^.]/gm)?.join('');
    let endMonth = (new Date(endDate)).toLocaleString('default', { month: 'short' }).match(/[^.]/gm)?.join('');

    let startDay = (new Date(startDate)).getDate(); 
    let endDay = (new Date(endDate)).getDate(); 
    
    let startYear = (new Date(startDate)).getFullYear(); 
    let endYear = (new Date(endDate)).getFullYear();
    let currentYear = (new Date()).getFullYear();

    let tariffString;
    if(tariff.toLocaleLowerCase().search(/подписка|пробный/gm) >= 0)
      tariffString = tariff.toLocaleLowerCase().match(/(?<=подписка |меню |пробный )\d+/gm)?.join('');
    if(tariff.toLocaleLowerCase().search(/базовый/gm) >= 0)
      tariffString = 30;
    
    return `${startDay}.${startMonth}${(startYear === currentYear) ? '' : `.${startYear}`} - ${endDay}.${endMonth}${(endYear === currentYear) ? '' : `.${endYear}`} (${tariffString ? (tariffString + '/') : ''}${diffDays})`;
  }
</script>

<template>
  <table>
    <tr>
      <th>ID</th>
      <th>ФИО</th>
      <th>Диеты</th>
      <th>Тарифы</th>
      <th>Адрес</th>
      <th>Номер</th>
      <th>Даты</th>
      <th>Оплата</th>
      <th>Комментарий курьеру</th>
      <th>Комментарий</th>
      <th v-if="sortingVariants[sortingState] === 'no'" class="status-header" @click="$event => {sortingState = (sortingState + 1) % 5}">Статус</th>
      <th v-if="sortingVariants[sortingState] === 'up'" class="status-header" @click="$event => {sortingState = (sortingState + 1) % 5}">Статус <CaretUp/></th>
      <th v-if="sortingVariants[sortingState] === 'down'" class="status-header" @click="$event => {sortingState = (sortingState + 1) % 5}">Статус <CaretDown/></th>
      <th v-if="sortingVariants[sortingState] === 'endtoday'" class="status-header" @click="$event => {sortingState = (sortingState + 1) % 5}">Статус (заканчивается сегодня)</th>
      <th v-if="sortingVariants[sortingState] === 'endtomorrow'" class="status-header" @click="$event => {sortingState = (sortingState + 1) % 5}">Статус (заканчивается завтра)</th>
    </tr>
    <tr :key="client.o_id" v-for="client in tableClients">
      <td class="id">{{ client.o_id }}</td>
      <td class="name">{{ client.client_name }}</td>
      <td>
        <template :key="index" v-for="(diet, index) in client.diets">
          {{ diet }}
          <hr v-if="index !== (client.diets.length - 1)">
        </template>
      </td>
      <td>
        <template :key="index" v-for="(tariff, index) in client.tariff">
          {{ tariff }}
          <hr v-if="index !== (client.tariff.length - 1)">
        </template>
      </td>
      <td>{{ client.address }}</td>
      <td>{{ client.phone }}</td>
      <td>
        <template :key="index" v-for="(date, index) in client.dates">
          {{ dateIntervalFromString(date, client.tariff[index]) }}
          <hr v-if="index !== (client.dates.length - 1)">
        </template>
      </td>
      <td :class="client.pay_status === 'Неоплачен ч.' ? 'unpayed' : 'payed'">
        Стоим.: {{ client.order_sum }} р.<br>
        {{ client.pay_status }}<br>
        <template v-if="client.pay_status === 'Неоплачен ч.'">
          Долг: {{ (client.order_sum - parseFloat(client.order_payed)).toFixed(2)}}
        </template>
      </td>
      <td>
        <div class="courier-comment">
          <p><TruckSvg/> {{ client.courier_comment }}</p>
        </div>
      </td>
      <td>
        <div class="inner-comment">
          <p><MessageSvg/> {{ client.inner_comment }}</p>
        </div>  
      </td>
      <td class="status">
        <template :key="index" v-for="(date, index) in client.dates">
          <p v-if="client.endsAt[index] < 0">
            закончилось {{ plural(Math.abs(client.endsAt[index]), '%d день', '%d дня', '%d дней') }} назад   
          </p>
          <p v-else-if="client.startsAt[index] < 0">
            заканчивается через {{ plural(client.endsAt[index], '%d день', '%d дня', '%d дней') }}
          </p>
          <p v-else>
            начинается через {{ plural(client.startsAt[index], '%d день', '%d дня', '%d дней') }}
          </p>
          <hr v-if="index !== (client.dates.length - 1)">
        </template>  
      </td>
    </tr>
  </table>
  <div class="empty-message" v-if="tableClients.length === 0">Нет соответстующих диет</div>
</template>

<style scoped>
  table, tr
  {
    width: 100%;
    border-collapse: collapse;
  }
  td, th
  {
    border: 1px solid #e5e8eb;
    text-align: center;
    max-width: 300px;
    padding: 5px 10px;
  }
  th{
    height: 60px;
  }
  .id
  {
    background-color: #99ff99;
  }
  .name
  {
    color: #4782be;
  }
  .unpayed
  {
    background-color: #ff9999;
  }
  .payed
  {
    background-color: #99ff99;
  }
  .courier-comment
  {
    border: 1px dashed #d7c2e1;
    text-align: start;
    padding: 5px;
  }
  .inner-comment
  {
    border: 1px dashed #f8bdbd;
    text-align: start;
    background-color: #f6ecec;
    padding: 5px;
  }
  tr:nth-child(odd) .status
  {
    background-color: #eaebff;
  }
  tr:nth-child(even) .status
  {
    background-color: #d3d6ff;
  }
  p
  {
    margin: 0;
  }
  tr:nth-child(even)
  {
    background-color: #f2f2f2;
  }
  hr
  {
    border-top: dashed 1px;
    background-color: transparent;
  }
  .status-header
  {
    cursor: pointer;
    width: 180px;
  }
  .empty-message
  {
    display: flex;
    widows: 100%;
    height: 100px;
    font-size: large;
    font-weight: bold;
    align-items: center;
    justify-content: center;
  }
</style>