<template>
  <div class="header">
    <div class="header-container">
      <div class="icon-box">
        <switch-camera />
        <switch-mirror />
      </div>
      <room-info />
      <end-control
        @on-destroy-room="onDestroyRoom"
        @on-exit-room="onExitRoom"
      />
    </div>
  </div>
</template>
<script setup lang="ts">
import EndControl from '../../RoomFooter/EndControl/index.vue';
import SwitchCamera from './SwitchCamera.vue';
import SwitchMirror from './SwitchMirror.vue';
import RoomInfo from './RoomInfo.vue';
import TUIRoomAegis from '../../../utils/aegis';

const emit = defineEmits(['log-out', 'on-destroy-room', 'on-exit-room']);

const onDestroyRoom = (info: { code: number; message: string }) => {
  emit('on-destroy-room', info);
  TUIRoomAegis.reportEvent({ name: 'destroyRoom', ext1: 'destroyRoom-success' });
};

const onExitRoom = (info: { code: number; message: string }) => {
  emit('on-exit-room', info);
  TUIRoomAegis.reportEvent({ name: 'exitRoom', ext1: 'exitRoom-success' });
};

</script>
<style scoped>
.header{
  height: 100%
}
.header-container{
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding:0 12px;
}
.icon-box{
   min-width: 50px;
   display: flex;
   align-items: center;
   justify-content: space-between
}
</style>
