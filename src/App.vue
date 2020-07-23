<template>
    <div id="app">
        <img alt="Vue logo" src="./assets/logo.png">
        <LineChart :graphDataList="graphDataList" />
    </div>
</template>

<script>
    import LineChart from "./components/LineChart";

    export default {
        name: 'App',
        components: {
            LineChart
        },
        data: function(){
            return {
                graphDataList: Array(30)
                    .fill()
                    .map((_, index) => {
                        return {
                            key: `2020-06-${index + 1}`,
                            docCount: Math.floor(Math.random() * 100)
                        }
                    })
            }
        }
    }
</script>

<style>
    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        margin-top: 60px;
    }
</style>
