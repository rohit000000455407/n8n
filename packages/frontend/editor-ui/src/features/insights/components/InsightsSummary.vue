<script setup lang="ts">
import { useI18n } from '@/composables/useI18n';
import { VIEWS } from '@/constants';
import {
	INSIGHT_IMPACT_TYPES,
	INSIGHTS_UNIT_IMPACT_MAPPING,
} from '@/features/insights/insights.constants';
import type { InsightsSummaryDisplay } from '@/features/insights/insights.types';
import type { InsightsSummary } from '@n8n/api-types';
import { smartDecimal } from '@n8n/utils/number/smartDecimal';
import { computed, ref, useCssModule } from 'vue';
import { useRoute } from 'vue-router';

const props = defineProps<{
	summary: InsightsSummaryDisplay;
	loading?: boolean;
}>();

const i18n = useI18n();
const route = useRoute();
const $style = useCssModule();

const lastNDays = ref(7);

const summaryTitles = computed<Record<keyof InsightsSummary, string>>(() => ({
	total: i18n.baseText('insights.banner.title.total'),
	failed: i18n.baseText('insights.banner.title.failed'),
	failureRate: i18n.baseText('insights.banner.title.failureRate'),
	timeSaved: i18n.baseText('insights.banner.title.timeSaved'),
	averageRunTime: i18n.baseText('insights.banner.title.averageRunTime'),
}));

const summaryWithRouteLocations = computed(() =>
	props.summary.map((s) => ({
		...s,
		to: { name: VIEWS.INSIGHTS, params: { insightType: s.id }, query: route.query },
	})),
);

const getImpactStyle = (id: keyof InsightsSummary, value: number) => {
	const impact = INSIGHTS_UNIT_IMPACT_MAPPING[id];
	if (value === 0 || impact === INSIGHT_IMPACT_TYPES.NEUTRAL) {
		return $style.neutral;
	}
	if (impact === INSIGHT_IMPACT_TYPES.POSITIVE) {
		return value > 0 ? $style.positive : $style.negative;
	}
	if (impact === INSIGHT_IMPACT_TYPES.NEGATIVE) {
		return value < 0 ? $style.positive : $style.negative;
	}
	return $style.neutral;
};
</script>

<template>
	<div :class="$style.insights">
		<N8nLoading v-if="loading" :class="$style.loading" :cols="5" />
		<ul v-else data-test-id="insights-summary-tabs">
			<li
				v-for="{ id, value, deviation, deviationUnit, unit, to } in summaryWithRouteLocations"
				:key="id"
				:data-test-id="`insights-summary-tab-${id}`"
			>
				<router-link :to="to" :exact-active-class="$style.activeTab">
					<strong>
						<N8nTooltip placement="bottom" :disabled="id !== 'timeSaved'">
							<template #content>
								{{ i18n.baseText('insights.banner.title.timeSaved.tooltip') }}
							</template>
							{{ summaryTitles[id] }}
						</N8nTooltip>
					</strong>
					<small :class="$style.days">{{
						i18n.baseText('insights.lastNDays', { interpolate: { count: lastNDays } })
					}}</small>
					<span v-if="value === 0 && id === 'timeSaved'" :class="$style.empty">
						<em>--</em>
						<small>
							<N8nTooltip placement="bottom">
								<template #content>
									<i18n-t keypath="insights.banner.timeSaved.tooltip">
										<template #link>
											<a href="#">{{
												i18n.baseText('insights.banner.timeSaved.tooltip.link.text')
											}}</a>
										</template>
									</i18n-t>
								</template>
								<N8nIcon :class="$style.icon" icon="info-circle" />
							</N8nTooltip>
						</small>
					</span>
					<span v-else>
						<em
							>{{ smartDecimal(value).toLocaleString('en-US') }} <i>{{ unit }}</i></em
						>
						<small v-if="deviation !== null" :class="getImpactStyle(id, deviation)">
							<N8nIcon
								:class="[$style.icon, getImpactStyle(id, deviation)]"
								:icon="deviation === 0 ? 'caret-right' : deviation > 0 ? 'caret-up' : 'caret-down'"
							/>
							<N8nTooltip placement="bottom" :disabled="id !== 'failureRate'">
								<template #content>
									{{ i18n.baseText('insights.banner.failureRate.deviation.tooltip') }}
								</template>
								{{ smartDecimal(Math.abs(deviation)).toLocaleString('en-US') }}{{ deviationUnit }}
							</N8nTooltip>
						</small>
					</span>
				</router-link>
			</li>
		</ul>
	</div>
</template>

<style lang="scss" module>
.insights {
	display: grid;
	grid-template-rows: auto 1fr;
	padding: var(--spacing-xs) 0 var(--spacing-2xl);

	ul {
		display: flex;
		height: 101px;
		align-items: stretch;
		justify-content: space-evenly;
		border: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
		border-radius: 6px;
		list-style: none;
		overflow-x: auto;

		li {
			display: flex;
			justify-content: stretch;
			align-items: stretch;
			flex: 1 0;
			border-left: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);

			&:first-child {
				border-left: 0;
			}
		}

		a {
			display: grid;
			align-items: center;
			align-content: center;
			width: 100%;
			height: 100%;
			padding: var(--spacing-3xs) var(--spacing-l) 0;
			border-bottom: 3px solid transparent;

			&:hover {
				background-color: var(--color-background-xlight);
				transition: background-color 0.3s;
			}

			&.activeTab {
				background-color: var(--color-background-xlight);
				border-color: var(--color-primary);
				transition: background-color 0.3s ease-in-out;
			}

			strong {
				justify-self: flex-start;
				color: var(--color-text-dark);
				font-size: var(--font-size-s);
				font-weight: 400;
				white-space: nowrap;
				margin-bottom: var(--spacing-3xs);
			}

			.days {
				padding: 0;
				margin: 0 0 var(--spacing-xs);
				color: var(--color-text-light);
				font-size: var(--font-size-2xs);
				font-weight: var(--font-weight-normal);
			}

			span {
				display: flex;
				align-items: baseline;

				&.empty {
					em {
						color: var(--color-text-lighter);
					}
					small {
						padding: 0;
						height: 21px;
						font-weight: var(--font-weight-bold);

						.icon {
							height: 20px;
							width: 8px;
							top: -3px;
							transform: translateY(0);
							color: var(--color-text-light);
						}
					}
				}
			}

			em {
				display: flex;
				align-items: baseline;
				justify-content: flex-start;
				color: var(--color-text-dark);
				font-size: 24px;
				line-height: 100%;
				font-weight: 600;
				font-style: normal;
				gap: var(--spacing-5xs);

				i {
					color: var(--color-text-light);
					font-size: 22px;
					font-style: normal;
				}
			}

			small {
				position: relative;
				display: flex;
				align-items: center;
				padding: 0 0 0 14px;
				margin: 0 0 0 var(--spacing-xs);
				font-size: var(--font-size-2xs);
				font-weight: var(--font-weight-bold);
				white-space: nowrap;
			}
		}
	}
}

.positive {
	color: var(--color-success);
}

.negative {
	color: var(--color-danger);
}

.neutral {
	color: var(--color-text-light);

	.icon {
		font-size: 17px;
	}
}

.icon {
	position: absolute;
	font-size: 17px;
	left: 0;
	top: 50%;
	transform: translateY(-50%);
}

.loading {
	display: flex;
	min-height: 91px;
	align-self: stretch;
	align-items: stretch;

	> div {
		margin: 0;
		height: auto;
	}
}
</style>
