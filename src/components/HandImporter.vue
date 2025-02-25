<template>
  <div class="mt-1 w-[52rem]">
    <textarea
      v-model="importText"
      :class="
        'block w-full h-[36rem] px-2 py-1 rounded-lg textarea text-sm font-mono ' +
        (importTextError ? 'textarea-error' : '')
      "
      @change="onImportTextChanged"
      spellcheck="false"
    />

    <div class="w-full">
      <button class="button-base button-blue mt-3 mr-3" @click="importHand">
        Import
      </button>
      <span v-if="importDoneText" class="mt-1">{{ importDoneText }}</span>
      <span v-if="importTextError" class="mt-1 text-red-500">
        {{ "Error: " + importTextError }}
      </span>
    </div>

    <div class="mt-2">
      Note: If the import has missing values, the current value will be used.
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from "vue";
import { Config, configKeys, useConfigStore, useStore } from "../store";
import {
  cardText,
  getExpectedBoardLength,
  parseCardString,
  Position,
  Result,
} from "../utils";
import { setRange, validateRange } from "../range-utils";
import * as invokes from "../invokes";

// eslint-disable-next-line @typescript-eslint/no-explicit-any
type NonValidatedHandJSON = any;
type HandJSON = {
  oopRange: string;
  ipRange: string;
  config: Omit<Config, "board"> & { board: string[] };
};

const INVALID_BOARD_ERROR = `Invalid '.config.board'. Set board cards manually for an example.`;

const CONFIG_INPUT_KEYS = configKeys.filter(
  (k) => ["expectedBoardLength"].indexOf(k) === -1
);

const config = useConfigStore();
const store = useStore();

const importText = ref("{\n  \n}");
const importTextError = ref("");
const importDoneText = ref("");

watch(
  () => store.ranges,
  () => generateImportText(),
  { deep: true }
);
watch(
  () => Object.values(config),
  () => generateImportText(),
  { deep: true }
);

const generateImportText = async () => {
  const configObj = Object.fromEntries(
    Object.entries(config).filter(
      ([key, _value]) => CONFIG_INPUT_KEYS.indexOf(key) !== -1
    )
  );

  const importObj = {
    oopRange: await invokes.rangeToString(Position.OOP),
    ipRange: await invokes.rangeToString(Position.IP),
    config: {
      ...configObj,
      board: config.board.map((cardId) => {
        const { rank, suitLetter } = cardText(cardId);
        return rank + suitLetter;
      }),
    },
  };

  importText.value = JSON.stringify(importObj, null, 2);
  importTextError.value = "";
  importDoneText.value = "";
};

const onImportTextChanged = () => {
  importDoneText.value = "";
  validateImportTextAndDisplayError();
};

const validateConfigPrimitives = (importConfig: unknown): Result => {
  if (typeof importConfig !== "object" || !importConfig)
    return {
      success: false,
      error: `Expected '.config' to be an object but got ${typeof importConfig}`,
    };

  for (const key of CONFIG_INPUT_KEYS.concat(Object.keys(importConfig))) {
    const newValue = (importConfig as Record<string, unknown>)[key];
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const existingValue = (config as unknown as Record<string, unknown>)[key];

    if (existingValue === undefined) {
      return { success: false, error: `Unexpected key '.config.${key}'` };
    }

    if (typeof existingValue === typeof newValue) continue;

    if (key === "board") return { success: false, error: INVALID_BOARD_ERROR };
    else
      return {
        success: false,
        error: `Expected '.config.${key}' to be ${typeof existingValue} but got ${typeof newValue}`,
      };
  }

  return { success: true };
};

const validateBoard = (board: unknown): Result => {
  if (!Array.isArray(board))
    return { success: false, error: INVALID_BOARD_ERROR };

  for (const i in board) {
    const card = board[i];
    if (typeof card !== "string") {
      return { success: false, error: INVALID_BOARD_ERROR };
    }

    const parsedCard = parseCardString(card);
    if (parsedCard === null) {
      return { success: false, error: INVALID_BOARD_ERROR };
    }
  }

  if (board.length > 5) {
    return { success: false, error: "Board cannot have more than 5 cards" };
  }

  return { success: true };
};

const parseJson = (json: string): Result<{ json?: NonValidatedHandJSON }> => {
  try {
    return { success: true, json: JSON.parse(json) };
  } catch (e) {
    const message = (e as Error).message;
    return { success: false, error: message };
  }
};

const validateImportTextAndDisplayError = (): Result<{
  json?: HandJSON;
}> => {
  importTextError.value = "";

  const validation = validateImportText();
  if (validation.success) return validation;

  importTextError.value = validation.error as string;
  return validation;
};

const validateImportText = (): Result<{ json?: HandJSON }> => {
  const parseValidation = parseJson(importText.value);
  if (!parseValidation.success) return parseValidation;

  const importJson = parseValidation.json;
  if (typeof importJson !== "object")
    return { success: false, error: "Not a valid JSON object" };

  const validateFns: (() => Result)[] = [
    () => validateConfigPrimitives(importJson.config),
    () => validateBoard(importJson.config?.board),
    () => validateRange(importJson.oopRange, "oopRange"),
    () => validateRange(importJson.ipRange, "ipRange"),
  ];
  for (const validate of validateFns) {
    const validation = validate();
    if (!validation.success) return validation;
  }

  const cfg = importJson.config;
  cfg.board = cfg.board.map(parseCardString);
  cfg.expectedBoardLength = getExpectedBoardLength(
    cfg.board.length,
    cfg.addedLines,
    cfg.removedLines
  );

  return { success: true, json: importJson as HandJSON };
};

const importHand = async () => {
  const validation = validateImportTextAndDisplayError();
  if (!validation.success) return;
  const importJson = validation.json as HandJSON;

  for (const key in importJson.config) {
    const newValue = (importJson.config as Record<string, unknown>)[key];
    (config as unknown as Record<string, unknown>)[key] = newValue;
  }

  await setRange(Position.OOP, importJson.oopRange, store);
  await setRange(Position.IP, importJson.ipRange, store);

  importDoneText.value = "Done!";
};

generateImportText();
</script>
