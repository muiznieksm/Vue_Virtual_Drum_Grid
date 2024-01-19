<template>
  <div id="DrumSequencer">
    <h1>Virtual Drum Grid</h1>
    <div class="rows">
      <div class="controls">
        <div class="buttons">
          <div class="controles">
            <button @click="play">{{ isPlaying ? "Stop" : "Play" }}</button>
            <button @click="reset">Reset</button>
            <button @click="clear">Clear</button>
          </div>
          <div class="save_func">
            <input
              type="text"
              v-model="layoutName"
              placeholder="Enter layout name"
            />
            <button id="save_button" @click="saveLayout">Save</button>
          </div>
        </div>
        <div class="tempo">
          {{ tempo }} bpm
          <input type="range" min="20" max="200" v-model="tempo" />
        </div>
      </div>

      <div v-for="instrument in instruments" :key="instrument.name" class="row">
        <div class="instrument-name">{{ instrument.name }}</div>
        <div
          v-for="step in steps"
          :key="step"
          class="step"
          :class="{
            'is-active': instrument.pattern[step],
            playhead: step === currentStep,
          }"
          @click="toggleStep(instrument, step)"
        ></div>
      </div>

      <div class="saved-layouts">
        <h2>Saved Layouts</h2>
        <div class="layouts-wrapper">
          <div
            v-for="layout in savedLayouts"
            :key="layout.id"
            class="saved-layout"
          >
            <div class="load" @click="loadLayout(layout)">
              {{ layout.name }}
              <span
                @click="() => handleDelete(layout)"
                class="material-symbols-outlined delete"
              >
                delete
              </span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { onMounted, ref, watch } from "vue";

export default {
  name: "DrumSequencer",
  setup() {
    const tempo = ref(120);
    const savedLayouts = ref([]);
    const layoutName = ref("");
    const steps = ref(Array.from({ length: 16 }, (_, i) => i));
    const instruments = ref([
      {
        name: "Open Hat",
        pattern: Array(16).fill(false),
        sound: "/drum-hit-open-hat-phonk.wav",
      },
      {
        name: "Closed Hat",
        pattern: Array(16).fill(false),
        sound: "/123.wav",
      },
      {
        name: "Clap",
        pattern: Array(16).fill(false),
        sound: "/drum-hit-open-hat-phonk.wav",
      },
      {
        name: "Kick",
        pattern: Array(16).fill(false),
        sound: "/hard-808-bass-slip_C.wav",
      },
      {
        name: "Kick",
        pattern: Array(16).fill(false),
        sound: "/hard-808-bass-slip_C.wav",
      },
    ]);

    const isPlaying = ref(false);
    const currentStep = ref(-1);
    let intervalId = null;

    watch(isPlaying, (newVal) => {
      if (!newVal && intervalId !== null) {
        clearInterval(intervalId);
        intervalId = null;
      }
    });

    const playSound = (soundPath) => {
      const audio = new Audio(soundPath);
      audio.play().catch((e) => {
        console.error("Playback failed", e);
      });
    };

    const play = () => {
      isPlaying.value = !isPlaying.value;
      if (isPlaying.value) {
        currentStep.value = -1;
        intervalId = setInterval(() => {
          advanceStep();
        }, (60 / tempo.value) * 1000);
      }
    };

    const advanceStep = () => {
      currentStep.value = (currentStep.value + 1) % steps.value.length;
      instruments.value.forEach((instrument) => {
        if (instrument.pattern[currentStep.value]) {
          playSound(instrument.sound);
        }
      });
    };

    const reset = () => {
      isPlaying.value = false;
      currentStep.value = -1;
    };

    const clear = () => {
      instruments.value.forEach((instrument) => {
        instrument.pattern.fill(false);
      });
    };

    const toggleStep = (instrument, step) => {
      instrument.pattern[step] = !instrument.pattern[step];
    };

    const fetchSavedLayouts = async () => {
      try {
        const response = await fetch("http://localhost:3000/layouts");
        savedLayouts.value = await response.json();
      } catch (error) {
        console.error("Failed to fetch saved layouts:", error);
      }
    };

    const saveLayout = async () => {
      if (!layoutName.value.trim()) {
        alert("Please enter a name for the layout.");
        return;
      }

      try {
        const response = await fetch("http://localhost:3000/layouts", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            name: layoutName.value,
            tempo: tempo.value,
            instruments: instruments.value.map((instr) => ({
              name: instr.name,
              pattern: instr.pattern,
            })),
          }),
        });

        if (!response.ok) {
          throw new Error("Failed to save layout");
        }

        const newLayout = await response.json();
        savedLayouts.value.push(newLayout);
        layoutName.value = "";

        fetchSavedLayouts();
      } catch (error) {
        console.error("Failed to save layout:", error);
      }
    };

    const loadLayout = (layout) => {
      tempo.value = layout.tempo;
      instruments.value.forEach((instrument, index) => {
        if (layout.instruments[index]) {
          instrument.pattern = layout.instruments[index].pattern;
        }
      });
    };

    const handleDelete = async (layout) => {
      if (
        !confirm(`Are you sure you want to delete the layout "${layout.name}"?`)
      ) {
        return;
      }
      try {
        const response = await fetch(
          `http://localhost:3000/layouts/${layout.id}`,
          {
            method: "DELETE",
          }
        );
        if (!response.ok) {
          throw new Error("Failed to delete layout");
        }
        savedLayouts.value = savedLayouts.value.filter(
          (l) => l.id !== layout.id
        );
      } catch (error) {
        console.error("Failed to delete layout:", error);
      }
    };

    onMounted(fetchSavedLayouts);

    return {
      tempo,
      steps,
      instruments,
      isPlaying,
      currentStep,
      play,
      reset,
      clear,
      toggleStep,
      savedLayouts,
      saveLayout,
      loadLayout,
      layoutName,
      handleDelete,
    };
  },
};
</script>

<style>
#app {
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: system-ui;
}
#DrumSequencer {
  user-select: none;
  display: flex;
  justify-content: center;
  flex-direction: column;
  align-items: center;
  margin-top: 50px;
  border: 4px solid #005cc8;
  padding: 20px;
  max-width: fit-content;
  border-radius: 5px;
  box-shadow: 5px 5px #005c;
  background-color: white;
}
.controls {
  margin-bottom: 20px;
  margin-left: 90px;
  width: 100%;
}

.tempo {
  width: 91.5%;
  display: flex;
  align-items: center;
  margin-top: 10px;
}

.tempo input {
  width: 80%;
  cursor: pointer;
}

.buttons {
  width: 91.5%;
  display: flex;
  justify-content: space-between;
  gap: 20px;
}

.buttons button {
  font-size: 18px;
  padding: 15px 30px;
  margin: 5px;
  background-color: #005cc8;
  color: #fff;
  border: none;
  border-radius: 8px;
  box-shadow: 5px 5px #005c;
  cursor: pointer;
  outline: none;
  transition: background-color 0.3s;
}

.buttons button:hover {
  background-color: #555;
}

.instrument-name {
  display: inline-block;
  width: 80px;
  margin-right: 10px;
  font-weight: 600;
}
.row {
  margin-bottom: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.step {
  display: inline-block;
  width: 60px;
  height: 60px;
  background-color: #005dc85e;
  margin-right: 2px;
  cursor: pointer;
  position: relative;
}
.step.is-active {
  background-color: #005c;
}
.step.playhead::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border-left: 2px solid #005cc8;
  z-index: 1;
}

.layouts-wrapper {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr 1fr 1fr;
  gap: 5px;
}

.saved-layouts {
  margin-top: 20px;
}

.saved-layout {
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  padding: 10px;
  border-radius: 4px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.saved-layout button {
  margin-left: 10px;
}

.controls input {
  font-size: 18px;
  padding-left: 10px;
  margin-right: 5px;
  border: 2px solid #ccc;
  border-radius: 4px;
  outline: none;
  height: 40px;
}

#save_button {
  box-shadow: 0px 0px #005c;
}

.save_func {
  display: flex;
  align-items: center;
}

.load {
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 2px;
}

.delete {
  color: red;
}
</style>
