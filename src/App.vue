<script setup lang="ts">
  import { ref, computed } from 'vue'
  import init, { list_all, clean } from "./external/chiritori-wasm/chiritori_wasm.js"

  const initialContent = `
async function main() {
  console.log('Hello, World! 1 [0]');
  /* < time-limited to="2020-12-31 23:59:59" > */
    console.log('🧹This code will be removed after 2020-12-31T23:59:59.999Z [1]');
  /* < /time-limited > */

  /* < removal-marker name="feature1" > */
    console.log('📌This code will be removed after feature1 released [2]');
  /* < /removal-marker > */

  /* < removal-marker name="feature2" > */
    console.log('📌This code will be removed after feature2 released [3]');
  /* < /removal-marker > */

  /* < removal-marker name="feature3" > */
    console.log('📌This code will be removed after feature3 released [4]');
  /* < /removal-marker > */

  /* < time-limited to="2099-12-31 23:59:59" > */
    console.log('📌This code will be removed after 2099-12-31T23:59:59.999Z [5]');
  /* < /time-limited > */

  /* =============== Nested tags =============== */

  /* < time-limited to="2020-12-31 23:59:59" > */
    console.log('🧹This code will be removed after 2020-12-31T23:59:59.999Z [6]');
    /* < time-limited to="2099-12-31 23:59:59" > */
      console.log('📌This code will be removed after 2099-12-31T23:59:59.999Z [7]');
    /* < /time-limited > */
  /* < /time-limited > */

  /* < time-limited to="2099-12-31 23:59:59" > */
    console.log('📌This code will be removed after 2099-12-31T23:59:59.999Z [8]');
    /* < time-limited to="2020-12-31 23:59:59" > */
      console.log('🧹This code will be removed after 2020-12-31T23:59:59.999Z [9]');
    /* < /time-limited > */
  /* < /time-limited > */

  /* =============== Unwrap Block =============== */

  const isReleased = await fetch('https://example.test/features/awesome-feature')

  /* < time-limited to="2020-12-31 23:59:59" unwrap-block > */
  if (isReleased) {
    console.log('📌This code is unconditionally executed after 2020-12-31T23:59:59.999Z [10]');
    const awesomeFeature = new awesomeFeature()
    awesomeFeature.run();
  }
  /* < /time-limited > */

  for (let i = 0; i < 10; i++) {
    /* < time-limited to="2020-12-31 23:59:59" unwrap-block > */
    if (isReleased) {
      console.log('📌This code is unconditionally executed after 2020-12-31T23:59:59.999Z [11]');
      awesomeFeature.add(i);
    }
    /* < /time-limited > */
  }

  /* =============== comment attribute =============== */

  /* < time-limited to="2001-12-31 23:59:59"
   * c="You can write your comments here."
   * > */
    console.log('[12]');
  /* < /time-limited > */

  /* < time-limited to="2020-12-31 23:59:59" unwrap-block
   * c="You can write your comments here."
   * > */
  if (isReleased) {
    console.log('[13]');
  }
  /* < /time-limited > */
}
`
  const content = ref(initialContent)
  const converted = ref("")
  const listHtml = ref("")
  const delimiterStart = ref("/* <")
  const delimiterEnd = ref("> */")
  const removalMarkerTargetsText = ref("feature1,feature2")
  const initialized = ref(false)
  const removalMarkerTargets = computed(() => removalMarkerTargetsText.value.split(',').filter(v => v.length > 0))

  const command = computed(() => {
    let command = `chiritori --delimiter-start='${delimiterStart.value}' --delimiter-end='${delimiterEnd.value}'`

    if (removalMarkerTargets.value.length > 0) {
      command += removalMarkerTargets.value.map(v => `--removal-marker-target-name='${v}'`).join(' ')
    }

    return command
  })

  init().then(() => initialized.value = true)

  const handleSubmit = () => {
    const configuration = {
      time_limited_configuration: {
        tag_name: "time-limited",
        time_offset: "+00:00",
        current: null
      },
      removal_marker_configuration: {
        tag_name: "removal-marker",
        targets: removalMarkerTargets.value
      }
    }
    const list = list_all(content.value, delimiterStart.value, delimiterEnd.value, configuration)
    converted.value = clean(content.value, delimiterStart.value, delimiterEnd.value, configuration)
    listHtml.value = list
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('\x1b[32m', '<span style="color: green">')
      .replaceAll('\x1b[31m', '<span style="color: red">')
      .replaceAll('\x1b[33m', '<span style="color: yellow">')
      .replaceAll('\x1b[0m', '</span>')
      .replaceAll('\n', '<br>')
  }
</script>

<template>
  <form class="app-boundary form-stack" v-if="initialized" @submit.prevent="handleSubmit">
    <div class="parameters">
      <label>
        Delimiter (Start)
        <input type="text" v-model="delimiterStart" />
      </label>
      <label>
        Delimiter (End)
        <input type="text" v-model="delimiterEnd" />
      </label>
      <label>
        &lt;removal-marker&gt; targets
        <input type="text" v-model="removalMarkerTargetsText" />
      </label>
      <div>
        <button class="convert-button">Convert</button>
      </div>
    </div>
    <div class="code-section form-vertical-stack form-vertical-stack--closed">
      <div class="form-vertical-stack__item">
        <textarea class="code-content" name="content" v-model="content" aria-label="Source Code"/>
      </div>
      <div class="form-vertical-stack__item">
        <div class="form-stack form-stack--closed">
          <div class="form-stack form-stack--closed form-stack--50">
            <div class="code-command">
              <code>
                $ cat /path/to/code.txt | {{ command }}
              </code>
            </div>
            <code class="code-converted">
              <div class="code-converted__body">{{ converted }}</div>
            </code>
          </div>
          <div class="form-stack form-stack--closed form-stack--50">
            <div class="code-command">
              <code>
                $ cat /path/to/code.txt | {{ command }} --list-all
              </code>
            </div>
            <code class="code-converted-list">
              <div class="code-converted-list__body" v-html="listHtml">
              </div>
            </code>
          </div>
        </div>
      </div>
    </div>
  </form>
</template>

<style scoped>
input[type="text"], textarea {
  padding: 0.5em;
  background: #111;
  color: gainsboro;
  line-height: 1.5em;
}

.app-boundary {
  height: 100%;
}

.parameters {
  display: flex;
  align-items: center;
  padding: 0 10px;

  @media screen and (max-width: 900px) {
    flex-direction: column;
    align-items: start;
  }
  
  label {
    padding: 10px;
    display: inline-block;
  }
}

.form-stack {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.form-stack--closed {
  gap: 0;
}

.form-vertical-stack {
  display: flex;
  flex-direction: row;
  gap: 1em;
  
  @media screen and (max-width: 900px) {
    flex-direction: column;
  }
}

.form-vertical-stack--closed {
  gap: 1px;
}

.code-command {
  background-color: #111;
  padding: 1em;
}

.code-section {
  padding: 1px 0;
  flex: 1;
  background-color: #768490;
}

.code-content {
  padding: 0.5em 2em;
  height: 100%;
  border: none;
  resize: none;

  @media screen and (max-width: 900px) {
    height: calc(100vh / 3);
    resize: vertical;
  }
}

.result-content {
  padding: 0.5em 2em;
  height: 100%;
  border: none;

  @media screen and (max-width: 900px) {
    min-height: calc(100vh / 3);
    height: 100%;
  }
}

.form-vertical-stack__item {
  width: 100%;
  height: 100%;

  * {
    width: 100%;
    box-sizing: border-box;
  }
}

.code-converted {
  height: 100%;
  padding: 1em;
  overflow-y: auto;
  background-color: #111;
  color: white;
}

.code-converted__body {
  box-sizing: border-box;
  line-height: 1.2em;
  white-space: pre;
  * {
    font-family: ui-monospace;
    text-transform: fill-width;
  }
}

.code-converted-list {
  height: 100%;
  padding: 1em;
  overflow-y: auto;
  background-color: #111;
  color: white;
}

.code-converted-list__body {
  box-sizing: border-box;
  line-height: 1.2em;
  white-space: pre;
  * {
    font-family: ui-monospace;
    text-transform: fill-width;
  }
}

.convert-button {
  padding: 10px 30px;
  cursor: pointer;
  color: gainsboro;
  background-color: #51565e;
  border-radius: 3px;
  border-color: #63646a #212123 #212123 #63646a;
}
</style>
