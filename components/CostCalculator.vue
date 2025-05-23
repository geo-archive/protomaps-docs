<script setup lang="ts">
import { ref, computed } from "vue";

const tileRequests = ref(10000000);

const formattedRequests = computed({
  get() {
    return tileRequests.value.toLocaleString();
  },

  set(value) {
    tileRequests.value = parseInt(value.replace(/,/g, ""), 10) || 0;
  },
});

const tilesPerView = ref(16);
const averageTileSizeKB = ref(100);
const cacheHitRate = ref(0.5);
const storedGB = ref(110);

const mapViews = computed(() => {
  return tileRequests.value / tilesPerView.value;
});

const hostedPerKViews = ref(3);

const outgoingGB = computed(() => {
  return (tileRequests.value * averageTileSizeKB.value) / 1000 / 1000;
});

const googleCost = computed(() => {
  const v = mapViews.value;
  if (v < 100000) return v * 0.007;
  else return 100000 * 0.007 + (v - 100000) * 0.0056;
});

const hostedCost = computed(() => {
  return (mapViews.value / 1000) * hostedPerKViews.value;
});

const cf = computed(() => {
  const obj = {
    workerInvocationCost: (tileRequests.value / 1000 / 1000) * 0.3,
    planCost: 5,
    storageRequestCost:
      (tileRequests.value / 1000 / 1000) * (1 - cacheHitRate.value) * 0.36,
    storageCost: storedGB.value * 0.015,
  };
  obj.total =
    obj.workerInvocationCost +
    obj.planCost +
    obj.storageRequestCost +
    obj.storageCost;
  return obj;
});

const aws = computed(() => {
  const obj = {
    cloudfrontGetRequestCost: (tileRequests.value / 10000) * 0.009,
    cloudfrontBandwidthCost: outgoingGB.value * 0.1,
    lambdaInvocationCost:
      (tileRequests.value / 1000 / 1000) * (1 - cacheHitRate.value) * 0.2,
    lambdaDurationCost:
      tileRequests.value * (1 - cacheHitRate.value) * 150 * 0.0000000067, // milliseconds, cost per 1ms on ARM
    s3GetObjectCost:
      (tileRequests.value / 1000) * (1 - cacheHitRate.value) * 0.0004,
    s3StorageCost: 0.023 * storedGB.value,
  };
  obj.total =
    obj.cloudfrontGetRequestCost +
    obj.cloudfrontBandwidthCost +
    obj.lambdaInvocationCost +
    obj.lambdaDurationCost +
    obj.s3GetObjectCost +
    obj.s3StorageCost;
  return obj;
});

const awsShow = ref(false);
const cfShow = ref(false);
</script>

<template>
  <div>
    <div>
      <div class="text-xl">
        <input id="requests" class="text-xl" v-model="formattedRequests" />
        <label for="requests"> monthly tile requests</label>
      </div>
      <div>
        <input
          aria-label="set the number of requests"
          type="range"
          id="range"
          min="0"
          step="1000000"
          v-model.number="tileRequests"
          max="100000000"
        />
      </div>
      <div>
        <strong>{{ mapViews.toLocaleString() }}</strong> monthly map viewer
        sessions
      </div>
      <div class="text-sm">
        <input id="tilesPerView" v-model.number="tilesPerView" />
        <label for="tilesPerView"> Tile loads per viewing session</label>
      </div>
      <div class="mt-4">
        <strong>{{ outgoingGB.toLocaleString() }} GB</strong> outgoing bandwidth
        to Internet
      </div>
      <div class="text-sm">
        <input id="averageTileSizeKB" v-model.number="averageTileSizeKB" />
        <label for="averageTileSizeKB"> Average tile size in kilobytes</label>
      </div>
      <div class="text-sm">
        <input v-model.number="cacheHitRate" /> CDN cache hit rate
      </div>
      <div class="text-sm">
        <input v-model.number="storedGB" /> Gigabytes stored
      </div>
    </div>
  </div>
  <div>
    <h3>Google Maps</h3>
    <span class="text-lg">{{ googleCost.toFixed(2) }} USD per month</span>
    <div class="text-sm">
      <a
        href="https://developers.google.com/maps/documentation/javascript/usage-and-billing"
        >SKU: Dynamic Maps</a
      >
    </div>
  </div>
  <div class="highlight">
    <h3>Hosted Map API</h3>
    <span class="text-lg">{{ hostedCost.toFixed(2) }} USD per month</span>
    <div class="text-sm">
      <input v-model.number="hostedPerKViews" /> USD per 1,000 sessions
    </div>
  </div>
  <div class="highlight">
    <h3>Protomaps on AWS</h3>
    <span class="text-lg"
      ><strong>{{ aws.total.toFixed(2) }} USD</strong> per month</span
    >
    <button class="showBreakdown" v-on:click="awsShow = !awsShow">
      {{ awsShow ? "Hide" : "Show" }} cost breakdown
    </button>
    <table v-show="awsShow">
      <thead>
        <tr>
          <th>Resource</th>
          <th>Unit Cost</th>
          <th>Multiplier</th>
          <th>Units</th>
          <th>Cost</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>CloudFront GET Requests</td>
          <td>
            <a href="https://aws.amazon.com/cloudfront/pricing/">
              0.009 / 10,000 (estimated)
            </a>
          </td>
          <td></td>
          <td>{{ tileRequests.toLocaleString() }}</td>
          <td>{{ aws.cloudfrontGetRequestCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>CloudFront Bandwidth to Internet</td>
          <td>
            <a href="https://aws.amazon.com/cloudfront/pricing/">
              0.10 / GB (estimated)
            </a>
          </td>
          <td></td>
          <td>{{ outgoingGB.toLocaleString() }} GB</td>
          <td>{{ aws.cloudfrontBandwidthCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>Lambda Function Invocations</td>
          <td>
            <a href="https://aws.amazon.com/lambda/pricing/">
              0.60 / 1,000,000
            </a>
          </td>
          <td>
            {{ 1 - cacheHitRate }} ({{ cacheHitRate * 100 }}% cache hit rate)
          </td>
          <td>
            {{ (tileRequests * (1 - cacheHitRate)).toLocaleString() }}
          </td>
          <td>{{ aws.lambdaInvocationCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>Lambda Compute</td>
          <td>
            <a href="https://aws.amazon.com/lambda/pricing/">
              0.0000000067 / 1ms
            </a>
          </td>
          <td>
            {{ 1 - cacheHitRate }} ({{ cacheHitRate * 100 }}% cache hit rate) *
            4 (512 MB RAM, avg. 200 ms duration)
          </td>
          <td>
            {{ (tileRequests * (1 - cacheHitRate) * 5).toLocaleString() }}
          </td>
          <td>{{ aws.lambdaDurationCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>S3 Bandwidth to CloudFront</td>
          <td>
            <a href="https://aws.amazon.com/cloudfront/pricing/"> 0.00 </a>
          </td>
          <td></td>
          <td>{{ (outgoingGB * (1 - cacheHitRate)).toFixed(2) }} GB</td>
          <td>0.00</td>
        </tr>
        <tr>
          <td>S3 GetObject Requests</td>
          <td>
            <a href="https://aws.amazon.com/s3/pricing/"> 0.0004 / 1000 </a>
          </td>
          <td>{{ 1 - cacheHitRate }}</td>
          <td>
            {{ (tileRequests * (1 - cacheHitRate)).toLocaleString() }}
          </td>
          <td>{{ aws.s3GetObjectCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>S3 Storage Costs</td>
          <td>
            <a href="https://aws.amazon.com/s3/pricing/"> 0.023 / GB </a>
          </td>
          <td></td>
          <td>{{ storedGB }} GB</td>
          <td>{{ aws.s3StorageCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td><strong>Total</strong></td>
          <td></td>
          <td></td>
          <td></td>
          <td>
            <strong>{{ aws.total.toFixed(2) }}</strong>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  <div>
    <h3>Protomaps on Cloudflare</h3>
    <span class="text-lg"
      ><strong>{{ cf.total.toFixed(2) }} USD</strong> per month</span
    >
    <button class="showBreakdown" v-on:click="cfShow = !cfShow">
      {{ cfShow ? "Hide" : "Show" }} cost breakdown
    </button>
    <table v-show="cfShow">
      <thead>
        <tr>
          <th>Resource</th>
          <th>Unit Cost</th>
          <th>Multiplier</th>
          <th>Units</th>
          <th>Cost</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Workers Invocations</td>
          <td>
            <a
              href="https://developers.cloudflare.com/workers/platform/pricing"
            >
              0.30 / million
            </a>
          </td>
          <td></td>
          <td>{{ tileRequests.toLocaleString() }}</td>
          <td>{{ cf.workerInvocationCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>Workers Paid Plan</td>
          <td>
            <a
              href="https://developers.cloudflare.com/workers/platform/pricing"
            >
              5.00
            </a>
          </td>
          <td></td>
          <td></td>
          <td>{{ cf.planCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>Bandwidth</td>
          <td>0.00</td>
          <td></td>
          <td>{{ outgoingGB }} GB</td>
          <td>0.00</td>
        </tr>
        <tr>
          <td>R2 GetObject Requests</td>
          <td>0.36 / million</td>
          <td>
            {{ 1 - cacheHitRate }} ({{ cacheHitRate * 100 }}% cache hit rate)
          </td>
          <td>
            {{ (tileRequests * (1 - cacheHitRate)).toLocaleString() }}
          </td>
          <td>{{ cf.storageRequestCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td>R2 Storage Costs</td>
          <td>0.015 / GB</td>
          <td>{{ storedGB }} GB</td>
          <td></td>
          <td>{{ cf.storageCost.toFixed(2) }}</td>
        </tr>
        <tr>
          <td><strong>Total</strong></td>
          <td></td>
          <td></td>
          <td></td>
          <td>
            <strong>{{ cf.total.toFixed(2) }}</strong>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<style scoped>
input {
  border: 1px solid #ccc;
  width: 2rem;
  border-radius: 3px;
  padding-left: 0.2rem;
}

.mt-4 {
  margin-top: 2rem;
}

.text-sm {
  font-size: 0.8rem;
}

.text-lg {
  font-size: 1.2rem;
}

.text-xl {
  font-size: 2rem;
}

.showBreakdown {
  display: block;
  cursor: pointer;
  font-weight: 500;
  color: var(--vp-c-brand-1);
  text-decoration: underline;
}

#range {
  margin-top: 1rem;
  margin-bottom: 1rem;
  width: 100%;
}

#requests {
  width: 15rem;
}
</style>
