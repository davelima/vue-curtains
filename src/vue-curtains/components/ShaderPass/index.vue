<template>
  <slot />
</template>

<script>
import { inject, onBeforeUnmount, watch, toRaw } from "vue";
import { useCurtains } from "../../hooks";
import { flattenDefaultParams } from "../../utils";
import { params } from "./params.js";
import { ShaderPass } from "curtainsjs";

export default {
  name: "ShaderPass",
  props: {
    uniqueKey: null,
    params: {
      type: Object,
      default: params,
    },
  },
  emits: [
    "before-create",
    "error",
    "loading",
    "ready",
    "after-resize",
    "render",
    "after-render",
    "before-remove",
  ],
  setup(props, { emit }) {
    let curtains;

    const renderTarget = inject("renderTarget", null);
    const parentRTKey = inject("uniqueKey", null);

    const key = props.uniqueKey || parentRTKey;

    const params = flattenDefaultParams(props.params);

    let shaderPass;

    watch(
      () => renderTarget,
      (newValue) => {
        if (newValue.value && !shaderPass) {
          createShaderPass();
        }
      },
      { deep: true }
    );

    const createShaderPass = () => {
      const rtReady = renderTarget
        ? renderTarget.value && renderTarget.value.renderer
        : true;

      if (curtains) {
        let existingShaderPass = [];
        if (key) {
          existingShaderPass = curtains.shaderPasses.filter(
            (pass) => pass._uniqueKey === key
          );
        }

        if (existingShaderPass.length) {
          shaderPass = existingShaderPass[0];
        } else if (rtReady) {
          emit("before-create");

          if (renderTarget && renderTarget.value) {
            params.renderTarget = toRaw(renderTarget.value);
          }

          shaderPass = new ShaderPass(curtains, params);

          if (key) {
            shaderPass._uniqueKey = key;
          }

          shaderPass
            .onError(() => emit("error", shaderPass))
            .onLoading((texture) => emit("loading", shaderPass, texture))
            .onReady(() => emit("ready", shaderPass))
            .onAfterResize(() => emit("after-resize", shaderPass))
            .onRender(() => emit("render", shaderPass))
            .onAfterRender(() => emit("after-render", shaderPass));
        }
      }
    };

    useCurtains((c) => {
      curtains = c;
      createShaderPass();
    });

    onBeforeUnmount(() => {
      if (shaderPass) {
        emit("before-remove", shaderPass);

        shaderPass.remove();
      }
    });

    watch(
      () => props.params.renderOrder,
      (newValue) => {
        shaderPass.setRenderOrder(newValue);
      }
    );
  },
};
</script>
