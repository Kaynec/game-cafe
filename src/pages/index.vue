<template>
  <q-layout>
    <q-page-container class="page">
      <q-page
        class="row items-center justify-evenly"
        style="max-width: 60rem; padding-inline: 1rem; margin: 0 auto"
      >
        <q-card class="fit q-pa-md">
          <q-form
            ref="authenticationForm"
            class="fit"
            @submit.prevent="onSubmit"
          >
            <div class="row items-center q-col-gutter-sm">
              <div class="col-12 col-md-5">
                <q-input
                  :rules="[(val:string) => !!val?.length || 'لطفا نام بازی را وارد کنید']"
                  dense
                  standout
                  label="عنوان بازی جدید"
                  v-model="newGameName.name"
                />
              </div>
              <div class="col-12 col-md-5">
                <q-input
                  dense
                  standout
                  type="number"
                  label="قیمت پیشفرض (برای هر ساعت)"
                  :rules="[(val:string) => !!val.length || 'لطفا نام بازی را وارد کنید']"
                  v-model="newGameName.basePrice"
                />
              </div>
              <div class="col-12 col-md-2 q-pb-xs q-mb-md">
                <q-btn
                  color="primary"
                  type="submit"
                  icon="add"
                  class="full-width"
                >
                  اضافه کردن
                </q-btn>
              </div>
            </div>
            <q-separator class="full-width q-mt-sm" />
          </q-form>
          <section class="game-container">
            <div
              v-for="item in gameNames"
              :key="item.id"
              style="max-width: 45rem; padding-top: 1rem"
            >
              <div style="display: flex; width: 100%; align-items: center">
                <span style="flex: 1">{{ item.name }}</span>

                <q-btn
                  color="primary"
                  @click="addNewTimeForGame(item)"
                  icon="add"
                >
                  شروع وقت جدید
                </q-btn>
                <q-btn
                  icon="delete"
                  flat
                  @click="removeGameFromGameName(item)"
                />
              </div>
              <div
                class="container q-mt-sm"
                style="display: flex; flex-direction: column; gap: 0.5rem"
              >
                <q-card
                  class="col q-ml-xs flex items-center rounded-borders q-py-sm q-px-sm shadow-0"
                  v-for="child in item.children"
                  :key="child.id"
                >
                  <div class="row q-col-gutter-lg">
                    <span>
                      <!-- Only the Time matters -->
                      زمان استارت تایم :{{
                        new Date(child.startTime)
                          .toLocaleDateString('fa-Fa', {
                            hour: 'numeric',
                            minute: 'numeric',
                            second: 'numeric'
                          })
                          .split(',')[1]
                      }}
                    </span>
                    <div>
                      هزینه تا الان
                      <span class="text-secondary text-bold">{{
                        child.currentPrice
                      }}</span>
                      تومان
                    </div>
                  </div>
                  <div style="margin-right: auto">
                    <q-btn
                      flat
                      icon="play_arrow"
                      v-if="child.isPaused"
                      @click="calculateNewTimeAgain(child)"
                    >
                    </q-btn>
                    <q-btn
                      flat
                      v-else
                      icon="stop"
                      @click="calculateTimeBetweenStartAndEnd(child, item)"
                    >
                    </q-btn>
                    <q-btn
                      icon="delete"
                      flat
                      @click="removeChildFromGame(child, item)"
                    />
                  </div>
                </q-card>
              </div>
            </div>
          </section>
        </q-card>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script setup lang="ts">
import { QForm, getCssVar, useQuasar } from 'quasar';
import { onMounted, ref } from 'vue';

interface Item {
  name: string;
  id: number;
  basePrice: number;
  children: Child[];
  priceForEachMinute: number;
}

interface Child {
  id: number;
  startTime: string | Date;
  endTime: string | Date;
  currentPrice: number;
  currentMinutesSpend: number;
  isPaused: boolean;
  isFinished: boolean;
}

type Items = Item[];

const $q = useQuasar();
const newGameName = ref({
  name: '',
  basePrice: 0
});

const gameNames = ref<Items>([]);

function setItemsToLocalStorage() {
  localStorage.setItem('gameNames', JSON.stringify(gameNames.value));
}

onMounted(() => {
  localStorage.getItem('gameNames') &&
    (gameNames.value = JSON.parse(localStorage.getItem('gameNames') || '[]'));

  window.addEventListener('beforeunload', () => setItemsToLocalStorage);
});

const authenticationForm = ref<QForm | null>();

function checkForDuplicates(gameNames: Items, newGameName: { name: string }) {
  const el = gameNames.find(el => el.name === newGameName.name);

  if (el) {
    $q.notify({
      type: 'error',
      color: 'negative',
      message: 'نام بازی تکراری می باشد'
    });
    return false;
  }
  return true;
}

function onSubmit() {
  authenticationForm.value?.validate().then(success => {
    if (!success) return;
    const canGoOn = checkForDuplicates(gameNames.value, newGameName.value);
    if (!canGoOn) return;
    gameNames.value.push({
      id: new Date().getTime() * 1526645,
      children: [],
      ...newGameName.value,
      priceForEachMinute: newGameName.value.basePrice / 60
    });
    newGameName.value.name = '';
    newGameName.value.basePrice = 0;
    authenticationForm.value?.reset();
    setItemsToLocalStorage();
  });
}

function removeGameFromGameName(item: Item) {
  gameNames.value = gameNames.value.filter(el => el.id !== item.id);
  setItemsToLocalStorage();
}

function addNewTimeForGame(item: Item) {
  item.children.push({
    id: new Date().getTime() * 1526645,
    startTime: new Date().toString(),
    endTime: '',
    currentPrice: 0,
    isPaused: false,
    isFinished: false,
    currentMinutesSpend: 0
  });

  setItemsToLocalStorage();
}

function calculateTimeBetweenStartAndEnd(child: Child, item: Item) {
  child.endTime = new Date();
  child.isPaused = true;
  child.startTime = new Date(child.startTime);
  const newMinutesSpent =
    (child.endTime.getTime() - child.startTime.getTime()) / 1000 / 60;
  child.currentMinutesSpend += newMinutesSpent;
  child.currentPrice = Math.floor(
    child.currentMinutesSpend * item.priceForEachMinute
  );
  setItemsToLocalStorage();
}

function calculateNewTimeAgain(child: Child) {
  child.isPaused = false;
  child.startTime = new Date(child.endTime);
  child.endTime = '';
  setItemsToLocalStorage();
}

function removeChildFromGame(child: Child, item: Item) {
  item.children = item.children.filter(el => el.id !== child.id);
  setItemsToLocalStorage();
}
</script>
<style scoped>
.page {
  background: url('../assets/game_cafe.jpeg');
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
  position: relative;
  width: 100%;
  height: 100%;
}
.page::before {
  background: rgba(0, 0, 0, 0.9);
  position: absolute;
  inset: 0;
  content: '';
}

.game-container {
  display: flex;
  flex-direction: column;
}
</style>
