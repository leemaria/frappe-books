<script lang="ts">
import { evaluateMathExpression, safeParseFloat } from 'utils/index';
import { defineComponent } from 'vue';
import Int from './Int.vue';

export default defineComponent({
  name: 'Float',
  extends: Int,
  computed: {
    inputType() {
      return 'number';
    },
  },
  methods: {
    parse(value: unknown): number {
      // Try to evaluate as math expression first if it's a string
      if (typeof value === 'string') {
        const evaluated = evaluateMathExpression(value);
        if (!isNaN(evaluated)) {
          return evaluated;
        }
      }
      return safeParseFloat(value);
    },
  },
});
</script>
