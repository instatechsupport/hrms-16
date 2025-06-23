<template>
	<div class="flex flex-col bg-white rounded w-full py-6 px-4 border-none">
	  <h2 class="text-lg font-bold text-gray-900">Hey, {{ employee?.data?.first_name }} 👋</h2>
  
	  <template v-if="settings.data?.allow_employee_checkin_from_mobile_app">
		<div class="font-medium text-sm text-gray-500 mt-1.5" v-if="lastLog">
		  Last {{ lastLogType }} was at {{ lastLogTime }}
		</div>
		<Button
		  class="mt-4 mb-1 drop-shadow-sm py-5 text-base"
		  id="open-checkin-modal"
		  @click="handleEmployeeCheckin"
		>
		  <template #prefix>
			<FeatherIcon
			  :name="nextAction.action === 'IN' ? 'arrow-right-circle' : 'arrow-left-circle'"
			  class="w-4"
			/>
		  </template>
		  {{ nextAction.label }}
		</Button>
	  </template>
  
	  <div v-else class="font-medium text-sm text-gray-500 mt-1.5">
		{{ dayjs().format("ddd, D MMMM, YYYY") }}
	  </div>
	</div>
  
	<ion-modal
	  v-if="settings.data?.allow_employee_checkin_from_mobile_app"
	  ref="modal"
	  trigger="open-checkin-modal"
	  :initial-breakpoint="1"
	  :breakpoints="[0, 1]"
	>
	  <div class="h-120 w-full flex flex-col items-center justify-center gap-5 p-4 mb-5">
		<div class="flex flex-col gap-1.5 mt-2 items-center justify-center">
		  <div class="font-bold text-xl">
			{{ dayjs(checkinTimestamp).format("hh:mm:ss a") }}
		  </div>
		  <div class="font-medium text-gray-500 text-sm">
			{{ dayjs().format("D MMM, YYYY") }}
		  </div>
		</div>
  
		<div class="w-full">
		  <label for="project" class="block text-sm font-medium text-gray-700">Select Project</label>
		  <select
			id="project"
			v-model="selectedProject"
			class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md"
		  >
			<option value="">Select a project</option>
			<option value="non-project">Non-Project (General Check-in)</option>
			<option v-for="project in projects.data" :key="project.name" :value="project">
			  {{ project.project_name }}
			</option>
		  </select>
		</div>
  
		<template v-if="settings.data?.allow_geolocation_tracking">
		  <span v-if="locationStatus" class="font-medium text-gray-500 text-sm">
			{{ locationStatus }}
		  </span>
  
		  <div v-if="distance !== null && selectedProject && selectedProject !== 'non-project'" class="text-sm text-gray-600">
			You are {{ distance.toFixed(2) }} meters away from the project location.
		  </div>
  
		  <div class="rounded border-4 translate-z-0 block overflow-hidden w-full h-170">
			<iframe
			  width="100%"
			  height="170"
			  frameborder="0"
			  scrolling="no"
			  marginheight="0"
			  marginwidth="0"
			  style="border: 0"
			  :src="`https://maps.google.com/maps?q=${latitude},${longitude}&hl=en&z=15&amp;output=embed`"
			>
			</iframe>
		  </div>
		</template>
  
		<Button variant="solid" class="w-full py-5 text-sm" @click="submitLog(nextAction.action)">
		  Confirm {{ nextAction.label }}
		</Button>
	  </div>
	</ion-modal>
  </template>
  
  <script setup>
  import { createResource, createListResource, toast, FeatherIcon } from "frappe-ui"
  import { computed, inject, ref, onMounted, onBeforeUnmount } from "vue"
  import { IonModal, modalController } from "@ionic/vue"
  
  const DOCTYPE = "Employee Checkin"
  
  const socket = inject("$socket")
  const employee = inject("$employee")
  const dayjs = inject("$dayjs")
  const checkinTimestamp = ref(null)
  const latitude = ref(0)
  const longitude = ref(0)
  const locationStatus = ref("")
  const selectedProject = ref(null)
  const distance = ref(null)
  
  const projects = createListResource({
	doctype: "Project",
	fields: ["name", "project_name", "custom_loc"],
	orderBy: "project_name asc",
	auto: true,
  })
  
  const settings = createResource({
	url: "hrms.api.get_hr_settings",
	auto: true,
  })
  
  const checkins = createListResource({
	doctype: DOCTYPE,
	fields: ["name", "employee", "employee_name", "log_type", "time", "device_id", "custom_projectn"], 
	filters: {
	  employee: employee.data.name,
	},
	orderBy: "time desc",
  })
  checkins.reload()
  
  const lastLog = computed(() => {
	if (checkins.list.loading || !checkins.data) return {}
	return checkins.data[0]
  })
  
  const lastLogType = computed(() => {
	return lastLog?.value?.log_type === "IN" ? "check-in" : "check-out"
  })
  
  const nextAction = computed(() => {
	return lastLog?.value?.log_type === "IN"
	  ? { action: "OUT", label: "Check Out" }
	  : { action: "IN", label: "Check In" }
  })
  
  const lastLogTime = computed(() => {
	const timestamp = lastLog?.value?.time
	const formattedTime = dayjs(timestamp).format("hh:mm a")
  
	if (dayjs(timestamp).isToday()) return formattedTime
	else if (dayjs(timestamp).isYesterday()) return `${formattedTime} yesterday`
	else if (dayjs(timestamp).isSame(dayjs(), "year"))
	  return `${formattedTime} on ${dayjs(timestamp).format("D MMM")}`
  
	return `${formattedTime} on ${dayjs(timestamp).format("D MMM, YYYY")}`
  })
  
  function calculateDistance(lat1, lon1, lat2, lon2) {
	const toRadians = (degree) => degree * (Math.PI / 180)
	const R = 6371e3
  
	const φ1 = toRadians(lat1)
	const φ2 = toRadians(lat2)
	const Δφ = toRadians(lat2 - lat1)
	const Δλ = toRadians(lon2 - lon1)
  
	const a =
	  Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
	  Math.cos(φ1) * Math.cos(φ2) * Math.sin(Δλ / 2) * Math.sin(Δλ / 2)
	const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  
	const distance = R * c
	return distance
  }
  
  function handleLocationSuccess(position) {
	latitude.value = position.coords.latitude
	longitude.value = position.coords.longitude
  
	locationStatus.value = `
	  Latitude: ${Number(latitude.value).toFixed(5)}°,
	  Longitude: ${Number(longitude.value).toFixed(5)}°
	`
  }
  
  function handleLocationError(error) {
	locationStatus.value = "Unable to retrieve your location"
	if (error) locationStatus.value += `: ERROR(${error.code}): ${error.message}`
  }
  
  const fetchLocation = () => {
	if (!navigator.geolocation) {
	  locationStatus.value = "Geolocation is not supported by your current browser"
	  return false
	} else {
	  locationStatus.value = "Locating..."
	  navigator.geolocation.getCurrentPosition(handleLocationSuccess, handleLocationError)
	  return true
	}
  }
  
  const handleEmployeeCheckin = () => {
	checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss")
  
	if (settings.data?.allow_geolocation_tracking) {
	  const locationAccessGranted = fetchLocation()
	  if (!locationAccessGranted) {
		toast({
		  title: "Error",
		  text: "Location access is required to check in.",
		  icon: "alert-circle",
		  position: "bottom-center",
		  iconClasses: "text-red-500",
		})
		return
	  }
	}
  }
  
  const submitLog = (logType) => {
	const action = logType === "IN" ? "Check-in" : "Check-out"
  
	// حالة Non-Project (لا توجد شروط)
	if (selectedProject.value === "non-project") {
	  checkins.insert.submit(
		{
		  employee: employee.data.name,
		  log_type: logType,
		  time: checkinTimestamp.value,
		  latitude: latitude.value,
		  longitude: longitude.value,
		  custom_projectn: null, // لا يوجد مشروع
		},
		{
		  onSuccess() {
			modalController.dismiss()
			checkins.reload()
			toast({
			  title: "Success",
			  text: `${action} successful (Non-Project)!`,
			  icon: "check-circle",
			  position: "bottom-center",
			  iconClasses: "text-green-500",
			})
			selectedProject.value = null
		  },
		  onError() {
			toast({
			  title: "Error",
			  text: `${action} failed!`,
			  icon: "alert-circle",
			  position: "bottom-center",
			  iconClasses: "text-red-500",
			})
		  },
		}
	  )
	  return
	}
  
	// حالة وجود مشروع (تطبق الشروط الأصلية)
	if (!selectedProject.value) {
	  toast({
		title: "Error",
		text: "Please select a project or Non-Project option to proceed.",
		icon: "alert-circle",
		position: "bottom-center",
		iconClasses: "text-red-500",
	  })
	  return
	}
  
	if (!latitude.value || !longitude.value) {
	  toast({
		title: "Error",
		text: "Location data is missing.",
		icon: "alert-circle",
		position: "bottom-center",
		iconClasses: "text-red-500",
	  })
	  return
	}
  
	try {
	  const geojson = JSON.parse(selectedProject.value.custom_loc)
	  if (!geojson.features || !geojson.features[0]?.geometry?.coordinates) {
		toast({
		  title: "Error",
		  text: "Invalid project coordinates.",
		  icon: "alert-circle",
		  position: "bottom-center",
		  iconClasses: "text-red-500",
		})
		return
	  }
  
	  const [projectLongitude, projectLatitude] = geojson.features[0].geometry.coordinates
	  const distance = calculateDistance(
		latitude.value,
		longitude.value,
		projectLatitude,
		projectLongitude
	  )
  
	  if (distance > 50) {
		toast({
		  title: "Error",
		  text: `You are ${distance.toFixed(2)} meters away from the project location. Maximum allowed distance is 50 meters.`,
		  icon: "alert-circle",
		  position: "bottom-center",
		  iconClasses: "text-red-500",
		})
		return
	  }
  
	  checkins.insert.submit(
		{
		  employee: employee.data.name,
		  log_type: logType,
		  time: checkinTimestamp.value,
		  latitude: latitude.value,
		  longitude: longitude.value,
		  custom_projectn: selectedProject.value.name,
		},
		{
		  onSuccess() {
			modalController.dismiss()
			toast({
			  title: "Success",
			  text: `${action} successful!`,
			  icon: "check-circle",
			  position: "bottom-center",
			  iconClasses: "text-green-500",
			})
			selectedProject.value = null
		  },
		  onError() {
			toast({
			  title: "Error",
			  text: `${action} failed!`,
			  icon: "alert-circle",
			  position: "bottom-center",
			  iconClasses: "text-red-500",
			})
		  },
		}
	  )
	} catch (error) {
	  toast({
		title: "Error",
		text: "Invalid project location data.",
		icon: "alert-circle",
		position: "bottom-center",
		iconClasses: "text-red-500",
	  })
	  console.error("Error parsing project location:", error)
	}
  }
  
  onMounted(() => {
	socket.emit("doctype_subscribe", DOCTYPE)
	socket.on("list_update", (data) => {
	  if (data.doctype == DOCTYPE) {
		checkins.reload()
	  }
	})
  })
  
  onBeforeUnmount(() => {
	socket.emit("doctype_unsubscribe", DOCTYPE)
	socket.off("list_update")
  })
  </script>