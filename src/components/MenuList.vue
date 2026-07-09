<template>
  <q-list separator>
    <q-item
      clickable
      v-ripple:purple
      v-for="items in data"
      :key="items"
      @click.stop="onClickItem(category, items)"
    >
      <q-item-section top avatar>
        <q-responsive style="width: 110px; height: 90px">
          <q-img
            :src="items.url_image"
            placeholder-src="placeholder.png"
            lazy
            fit="scale-down"
            class="radius8"
            spinner-color="primary"
            spinner-size="sm"
          />
        </q-responsive>
      </q-item-section>
      <q-item-section>
        <q-item-label>
          <div
            class="text-weight-medium text-subtitle2"
            v-html="items.item_name"
          ></div>
        </q-item-label>
        <q-item-label>
          <div
            class="text-grey ellipsis-2-lines text-caption line-normal"
            v-html="items.item_description"
          ></div>

          <template v-if="items.promo_data?.message && items.available">
            <q-badge
              :color="isEligible(items) ? 'green-1' : 'orange-1'"
              :text-color="isEligible(items) ? 'green-5' : 'orange-5'"
              rounded
              multi-line
            >
              {{ getPromoMessage(items) }}
            </q-badge>
          </template>

          <div class="q-mt-xs q-mb-xs q-gutter-x-sm">
            <template v-for="dish in items.dish_list" :key="dish">
              <q-avatar size="2em" class="bg-yellow-9">
                <img :src="dish.url_image" />
              </q-avatar>
            </template>
          </div>

          <div
            class="text-weight-bold text-overline letter-spacing-none"
            :class="{ 'text-green': isEligible(items) }"
          >
            <template v-if="isEligible(items)">
              {{ $t("Free!") }}
            </template>
            <template v-else>
              <span
                v-if="items.lowest_price_discount_raw > 0"
                class="text-strike text-red"
                >{{ items.lowest_price_discount }}</span
              >
              {{ items.lowest_price }}
            </template>

            <div
              v-if="items.lowest_price_discount_raw <= 0 && items.has_discount"
              class="text-blue-8"
            >
              {{ $t("Check Offers") }}
            </div>
          </div>

          <q-badge
            v-if="!items.available"
            color="disabled"
            text-color="disabled"
            :label="$t('Not available')"
            rounded
          />

          <q-tooltip
            v-model="item_tooltip[`${items.item_uuid}${category.cat_id}`]"
            :no-parent-event="true"
          >
            {{ items.item_unavailable }}</q-tooltip
          >
        </q-item-label>
      </q-item-section>
      <q-item-section side class="row items-stretch">
        <div class="column items-center col-12 q-gutter-y-sm">
          <div class="col">
            <FavsItem
              ref="favs"
              :layout="3"
              :item_token="items.item_uuid"
              :cat_id="category.cat_id"
              :active="items.is_favorite"
              :data="items"
              @after-savefav="afterSavefavItem"
              @on-saved="onSavedfavItem"
            />
          </div>
          <div class="col" v-if="items.total_allergens > 0">
            <q-btn
              round
              unelevated
              color="mygrey"
              text-color="dark"
              size="sm"
              icon="o_info"
              @click.stop="showAllergens(this.merchant_id, items.item_id)"
            />
          </div>
          <div class="col">
            <template v-if="findItem(items.item_uuid, category.cat_id)">
              <q-btn
                outline
                color="primary"
                round
                size="sm"
                @click.stop="
                  showOptions(findItem(items.item_uuid, category.cat_id))
                "
              >
                <div style="font-size: 14px">
                  {{ findItemQty(items.item_uuid, category.cat_id) }}
                </div>
              </q-btn>
            </template>
            <template v-else>
              <q-btn
                round
                unelevated
                color="primary"
                text-color="white"
                size="sm"
                icon="las la-plus"
                :disable="!items.available || !isEligibleItem(items)"
                @click.stop="onClickItem(category, items)"
              />
            </template>
          </div>
        </div>
      </q-item-section>
    </q-item>
  </q-list>
  <!-- end item list -->
</template>

<script>
import { defineAsyncComponent } from "vue";
import { useCartStore } from "stores/CartStore";
import { useDataStore } from "stores/DataStore";
import APIinterface from "src/api/APIinterface";

export default {
  name: "MenuList",
  props: ["data", "category", "merchant_id", "promoEligibility"],
  components: {
    FavsItem: defineAsyncComponent(() => import("src/components/FavsItem.vue")),
  },
  setup() {
    const CartStore = useCartStore();
    const DataStore = useDataStore();
    return { CartStore, DataStore };
  },
  data() {
    return {
      item_tooltip: {},
    };
  },
  methods: {
    getPromoMessage(item) {
      return (
        this.promoEligibility[item.item_id]?.message ||
        item?.promo_data?.message
      );
    },
    isEligible(item) {
      return this.promoEligibility[item.item_id]?.is_eligible;
    },
    isEligibleItem(item) {
      if (!item.is_promo_free_item) {
        return true;
      }
      return this.promoEligibility[item.item_id]?.is_eligible || false;
    },
    onSavedfavItem(data, found) {
      data.is_favorite = found;
      // CLEAR HOME PAGE FILTER AND FAV
      this.DataStore.feed_filter = [];
      this.DataStore.fav_saved_data = null;
    },
    onClickItem(category, items) {
      console.log("onClickItem", items);
      const isEligibleItem = this.isEligibleItem(items);
      console.log("isEligibleItem", isEligibleItem);
      if (!isEligibleItem) {
        const message = this.getPromoMessage(items); //items?.promo_data?.message || "";
        APIinterface.ShowAlert(message, this.$q.capacitor, this.$q);
        return;
      }

      if (!items.available) {
        this.item_tooltip[`${items.item_uuid}${category.cat_id}`] = true;
        setTimeout(() => {
          this.item_tooltip[`${items.item_uuid}${category.cat_id}`] = false;
        }, 2000);
        return false;
      }
      const find_item = this.findItem(items.item_uuid, category.cat_id);
      if (find_item) {
        this.showOptions(find_item);
        return;
      } else {
        this.$emit("onClickitems", {
          cat_id: category.cat_id,
          item_uuid: items.item_uuid,
        });
      }
    },
    showAllergens(merchant_id, item_id) {
      this.$emit("showAllergens", {
        merchant_id: merchant_id,
        item_id: item_id,
      });
    },
    showOptions(value) {
      this.$emit("showOptions", value);
    },
    findItem(value, cat_id) {
      const items = this.CartStore.getItems;
      if (!items || items.length === 0) {
        return false;
      }
      const foundItem = items.filter(
        (item) => item.item_token === value && item.cat_id == cat_id
      );
      return foundItem.length > 0 ? foundItem : false;
    },
    findItemQty(value, cat_id) {
      const items = this.CartStore.getItems;
      if (!items || items.length === 0) {
        return false;
      }
      const summedQty = items
        .filter((item) => item.item_token === value && item.cat_id == cat_id)
        .reduce((acc, item) => acc + item.qty, 0);

      return summedQty;
    },
  },
};
</script>
