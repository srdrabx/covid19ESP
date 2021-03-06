<template>
    <div class="flex w-full xl:w-1/2 lg:pl-2">
        <div class="flex flex-wrap w-full shadow-lg bg-gray-800 rounded-lg">
            <div class="box-header text-sm">
                <div class="">
                    {{ chartTitle }}
                </div>
                <div class="cursor-pointer" v-if="show">
                    <div class="relative custom-select">
                        <select v-model="countrySelected">
                            <template v-for="ccaa in countries">
                                <option :value="ccaa.iso2" :key="ccaa.iso2">
                                    {{ ccaa.country }}
                                </option>
                            </template>
                        </select>
                    </div>
                </div>
            </div>

            <div class="box-body ">
                <div class="w-full h-64">
                    <div v-if="!show" class="w-full h-full flex justify-center items-center">
                        <vue-loaders
                            name="ball-scale"
                            color="#90CDF4"
                            scale="1.2"
                        />
                    </div>
                    <chart
                        v-if="show"
                        :key="'country_chart_'+iso"
                        width="100%"
                        height="100%"
                        type="line"
                        :options="options"
                        :series="series"
                    />
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import { mapState, mapGetters } from 'vuex'
import _ from 'lodash'
import VueApexCharts from 'vue-apexcharts'
var es = require("apexcharts/dist/locales/es.json")
import 'vue-loaders/dist/vue-loaders.css';
import VueLoaders from 'vue-loaders';

export default {
    name: 'CountryCharts',
    components: {
        'chart': VueApexCharts,
        'vue-loaders': VueLoaders.component
    },
    props: {
        iso: {
            required: false,
            default: 'all'
        }
    },
    data: () => ({
        show: false,
        now: parseInt(Date.now()),
    }),
    computed: {
        ...mapState({
            info: state => state.data,
            totals: state => state.totals,
            daily: state => state.daily,
            countries: state => state.countries,
        }),

        ...mapGetters(['confirmedByCountry', 'nowByCountry']),

        countrySelected: {
            get() {
                return this.iso
            },
            set(val) {
                this.$emit('country', val)
            }
        },

        country() {
            if (this.info) {
                if (this.iso == 'all') {
                    return this.daily
                }
                
                return this.confirmedByCountry(this.iso)
            }

            return false
        },

        latest() {
            if (this.info) {
                if (this.iso != 'all') {
                    return this.nowByCountry(this.iso)
                }
            }

            return false
        },

        series() {

            if (this.country && this.confirmed && this.deaths && this.recovered) {
                let confirmed = _.map(this.confirmed, (value, key) => ({
                    x: parseInt(key),
                    y: value
                }))

                let deaths = _.map(this.deaths, (value, key) => ({
                    x: parseInt(key),
                    y: value
                }))

                // let recovered = _.map(this.recovered, (value, key) => ({
                //     x: parseInt(key),
                //     y: value
                // }))

                return [
                    {
                        name: 'Infectados',
                        data: _.sortBy(confirmed, 'x')
                    },
                    // {
                    //     name: 'Recuperados',
                    //     data: _.sortBy(recovered, 'x')
                    // },
                    {
                        name: 'Muertes',
                        data: _.sortBy(deaths, 'x')
                    }
                ]
            } 

            return []
        },

        options() {

            return {
                chart: {
                    id: 'by_country',
                    width: 'auto',
                    height: '100%',
                    type: 'line',
                    foreColor: '#fff',
                    toolbar: {
                        show: false
                    },
                    locales: [es],
                    defaultLocale: 'es',
                },
                // title: {
                //     text: this.chartTitle,
                //     align: 'left'
                // },
                colors: ['#FAF089', '#48BB78', '#F56565'],
                grid: {
                    borderColor: "#40475D",
                },
                stroke: {
                    curve: 'straight',
                    width: 3
                },
                xaxis: {
                    type: 'datetime',
                },
                tooltip: {
                    theme: 'dark',
                    x: {
                        formatter: (title) => new Date(title).toLocaleDateString('es-ES', { year: 'numeric', month: 'long', day: 'numeric' }),
                        title: {
                            formatter: (title) => new Date(title).toLocaleDateString('es-ES', { year: 'numeric', month: 'long', day: 'numeric' }),
                        }
                    }
                },
                dataLabels: {
                    enabled: false
                },
                legend: {
                    show: true,
                    floating: true,
                    horizontalAlign: 'right',
                    onItemClick: {
                        toggleDataSeries: true
                    },
                    position: 'top',
                    offsetY: -5,
                },
            }
        },

        confirmed() {
            if (this.country) {
                let data;

                if (this.iso == 'all') {
                    data = this.country.confirmed
                } else {
                    data = this.country.confirmed.history
                }

                let confirmed = _.mapKeys(data, (value, key) => {
                    return parseInt(new Date(key).getTime())
                })

                if (this.iso != 'CN' && this.latest != false) {
                    this.$set(confirmed, this.now, this.latest.confirmed)
                }

                return confirmed
            }

            return false
        },

        deaths() {
            if (this.country) {
                let data;

                if (this.iso == 'all') {
                    data = this.country.deaths
                } else {
                    data = this.country.deaths.history
                }

                let deaths = _.mapKeys(data, (value, key) => {
                    return parseInt(new Date(key).getTime())
                })

                if (this.iso != 'CN') {
                    this.$set(deaths, this.now, this.latest.deaths)
                }

                return deaths
            }

            return false
        },

        recovered() {
            if (this.country) {

                let data;

                if (this.iso == 'all') {
                    data = this.country.recovered
                } else {
                    if (_.has(this.country.recovered, 'history')) {
                        data = this.country.recovered.history    
                    }
                }

                if (data) {
                    let recovered =  _.mapKeys(data, (value, key) => {
                        return parseInt(new Date(key).getTime())
                    });

                    if (this.iso != 'CN') {
                        this.$set(recovered, this.now, this.latest.recovered)
                    }

                    return recovered
                }

                return []

                
            }

            return false
        },

        chartTitle() {
            return 'Datos del virus por país'
        },

        isChartReady() {

            if (this.latest != false &&
                this.country != false &&
                this.confirmed != false &&
                this.deaths != false &&
                this.recovered != false) {

                return true
            }

            return false
        }
    },

    watch: {

        info: {
            immediate: true,
            handler (value) {
                if (value == null) {
                    this.show = false 
                }
            }
        },

        isChartReady(value) {
            if (value) {
                this.showChartFn()
            }
        },

        series(value) {
            if (value) {
                this.showChartFn()
            }
        },

        iso: {
            immediate: true,
            handler (value) {
                if (value) {
                    this.countrySelected = value
                    this.showChartFn()
                }
            }
        }
    },

    mounted() {
        //
    },
    methods: {
        showChartFn() {
            if (this.info != null) {
                this.$nextTick(() => {
                    setTimeout(() => {
                        this.show = true
                    }, 500)
                })
            }
        }
    }
}
</script>
