<template>
	<header
		class="sticky flex items-center justify-between top-0 z-10 border-b bg-surface-white px-3 py-2.5 sm:px-5"
	>
		<Breadcrumbs :items="breadcrumbs" />

		<div v-if="user.data?.is_moderator" class="flex gap-2 mt-2">
		<a href="https://www.mari.petconlms.com/app">
			<Button variant="outline">
				{{ __('Go to Desk') }}
			</Button>
		</a>

		<router-link :to="{ name: 'CourseForm', params: { courseName: 'new' } }">
			<Button variant="solid">
				<template #prefix>
					<Plus class="h-4 w-4 stroke-1.5" />
				</template>
				{{ __('Create new Course') }}
			</Button>
		</router-link>
	</div>
	</header>
	<div class="p-5 pb-10">
		<div
			class="flex flex-col lg:flex-row space-y-4 lg:space-y-0 lg:items-center justify-between mb-5"
		>
			<div class="text-lg font-semibold">
				{{ __('All Courses') }}
			</div>
			<div
				class="flex flex-col space-y-2 lg:space-y-0 lg:flex-row lg:items-center lg:space-x-4"
			>
				<TabButtons
					v-if="user.data"
					:buttons="courseTabs"
					v-model="currentTab"
				/>

				<div class="grid grid-cols-2 gap-2">
					<FormControl
						v-model="title"
						:placeholder="__('Search by Title')"
						type="text"
						class="min-w-40 lg:min-w-0 lg:w-32 xl:w-40"
						@input="updateCourses()"
					/>
					<div class="min-w-40 lg:min-w-0 lg:w-32 xl:w-40">
						<Select
							v-if="categories.length"
							v-model="currentCategory"
							:options="categories"
							:placeholder="__('Category')"
							@change="updateCourses()"
						/>
					</div>
				</div>
			</div>
		</div>
		<div
			v-if="courses.data?.length"
			class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4 gap-5"
		>
			<router-link
				v-for="course in courses.data"
				:to="{ name: 'CourseDetail', params: { courseName: course.name } }"
			>
				<CourseCard :course="course" />
			</router-link>
		</div>
		<div
			v-else-if="!courses.list.loading"
			class="flex flex-col items-center justify-center text-sm text-ink-gray-5 italic mt-48"
		>
			<BookOpen class="size-10 mx-auto stroke-1 text-ink-gray-4" />
			<div class="text-lg font-medium mb-1">
				{{ __('No courses found') }}
			</div>
			<div class="leading-5 w-2/5 text-center">
				{{
					__(
						'There are no courses matching the criteria. Keep an eye out, fresh learning experiences are on the way soon!'
					)
				}}
			</div>
		</div>
		<div
			v-if="!courses.list.loading && courses.hasNextPage"
			class="flex justify-center mt-5"
		>
			<Button @click="courses.next()">
				{{ __('Load More') }}
			</Button>
		</div>
	</div>
</template>
<script setup>
import {
	Breadcrumbs,
	Button,
	createListResource,
	FormControl,
	Select,
	TabButtons,
} from 'frappe-ui'
import { computed, inject, onMounted, ref, watch } from 'vue'
import { BookOpen, Plus } from 'lucide-vue-next'
import { updateDocumentTitle } from '@/utils'
import CourseCard from '@/components/CourseCard.vue'

const user = inject('$user')
const dayjs = inject('$dayjs')
const start = ref(0)
const pageLength = ref(30)
const categories = ref([])
const currentCategory = ref(null)
const title = ref('')
const certification = ref(false)
const filters = ref({})
const currentTab = ref('Catalogue')

onMounted(() => {
	setFiltersFromQuery()
	updateCourses()
	categories.value = [
		{
			label: '',
			value: null,
		},
	]
})

const setFiltersFromQuery = () => {
	let queries = new URLSearchParams(location.search)
	title.value = queries.get('title') || ''
	currentCategory.value = queries.get('category') || null
	certification.value = queries.get('certification') || false
}

const courses = createListResource({
	doctype: 'LMS Course',
	url: 'lms.lms.utils.get_courses',
	cache: ['courses', user.data?.name],
	pageLength: pageLength.value,
	start: start.value,
	onSuccess(data) {
		let allCategories = data.map((course) => course.category)
		allCategories = allCategories.filter(
			(category, index) => allCategories.indexOf(category) === index && category
		)
		if (categories.value.length <= allCategories.length) {
			updateCategories(data)
		}
	},
})

const updateCourses = () => {
	updateFilters()
	courses.update({
		filters: filters.value,
	})
	courses.reload()
}

const updateFilters = () => {
	updateCategoryFilter()
	updateTitleFilter()
	updateCertificationFilter()
	updateTabFilter()
	updateStudentFilter()
	setQueryParams()
}

const updateCategoryFilter = () => {
	if (currentCategory.value) {
		filters.value['category'] = currentCategory.value
	} else {
		delete filters.value['category']
	}
}

const updateTitleFilter = () => {
	if (title.value) {
		filters.value['title'] = ['like', `%${title.value}%`]
	} else {
		delete filters.value['title']
	}
}

const updateCertificationFilter = () => {
	if (certification.value) {
		filters.value['certification'] = 1
	} else {
		delete filters.value['certification']
	}
}

const updateTabFilter = () => {
	if (!user.data) {
		return
	}

	delete filters.value['Catalogue']
	delete filters.value['created']
	delete filters.value['published_on']
	delete filters.value['upcoming']

	if (currentTab.value == 'Enrolled' && user.data?.is_student) {
		filters.value['enrolled'] = 1
		delete filters.value['published']
	} else {
		delete filters.value['published']
		delete filters.value['enrolled']

		if (currentTab.value == 'Catalogue') {
			filters.value['published'] = 1
			filters.value['upcoming'] = 0
			filters.value['Catalogue'] = 1
		} else if (currentTab.value == 'Upcoming') {
			filters.value['upcoming'] = 1
			filters.value['published'] = 1
		} else if (currentTab.value == 'Created') {
			filters.value['created'] = 1
		}
	}
}

const updateStudentFilter = () => {
	if (!user.data || (user.data?.is_student && currentTab.value != 'Enrolled')) {
		filters.value['published'] = 1
	}
}

const setQueryParams = () => {
	let queries = new URLSearchParams(location.search)
	let filterKeys = {
		title: title.value,
		category: currentCategory.value,
		certification: certification.value,
	}

	Object.keys(filterKeys).forEach((key) => {
		if (filterKeys[key]) {
			queries.set(key, filterKeys[key])
		} else {
			queries.delete(key)
		}
	})

	history.replaceState({}, '', `${location.pathname}?${queries.toString()}`)
}

const updateCategories = (data) => {
	data.forEach((course) => {
		if (
			course.category &&
			!categories.value.find((category) => category.value === course.category)
		)
			categories.value.push({
				label: course.category,
				value: course.category,
			})
	})
}

watch(currentTab, () => {
	updateCourses()
})

const courseType = computed(() => {
	let types = [
		{ label: __(''), value: null },
		{ label: __('New'), value: 'New' },
		{ label: __('Upcoming'), value: 'Upcoming' },
	]
	if (user.data?.is_student) {
		types.push({ label: __('Enrolled'), value: 'Enrolled' })
	} else {
		types.push({ label: __('Created'), value: 'Created' })
	}
	return types
})

const courseTabs = computed(() => {
	let tabs = [
		{
			label: __('Catalogue'),
		},
		{
			label: __('Upcoming'),
		},
	]
	if (user.data?.is_student) {
		tabs.push({ label: __('Enrolled') })
	} else {
		tabs.push({ label: __('Created') })
	}
	return tabs
})

const breadcrumbs = computed(() => [
	{
		label: __('Courses'),
		route: { name: 'Courses' },
	},
])

const pageMeta = computed(() => {
	return {
		title: 'Courses',
		description: 'All published courses.',
	}
})

updateDocumentTitle(pageMeta)
</script>
