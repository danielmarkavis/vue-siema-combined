<template>
    <div>
        <slider :options="options" @change="onChange" @init="onInit" ref="siema">
		<img src="https://via.placeholder.com/1280x768" alt="placeholder image" />
		<img src="https://via.placeholder.com/1280x768" alt="placeholder image" />
		<img src="https://via.placeholder.com/1280x768" alt="placeholder image" />
		<img src="https://via.placeholder.com/1280x768" alt="placeholder image" />
        </slider>
        <div class="btn" @click="$refs.siema.prev()">Prev</div>
        <div class="btn" @click="$refs.siema.next()">Next</div>
    </div>
</template>

<script>
    export default {
        name: "Image Gallery",
        components: {
            'slider': require('@/components/Slider').default,
        },
        data() {
            return {
                options: {
                    duration: 200,
                    easing: 'ease-out',
                    perPage: 1,
                    startIndex: 0,
                    draggable: true,
                    multipleDrag: true,
                    threshold: 20,
                    loop: true,
                    rtl: false,
                }
            }
        },
	methods: {
		onInit() {
			console.log('init');
		},
		onChange() {
			console.log('change');
		},
	}
    };
</script>