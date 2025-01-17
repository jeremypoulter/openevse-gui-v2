<script>
	import RemovableTag from "./../../ui/RemovableTag.svelte";
	import Borders from "./../../ui/Borders.svelte";
	import SelectFile from "./../../ui/SelectFile.svelte";
	import AlertBox from "./../../ui/AlertBox.svelte";
	import { status_store } from "./../../../lib/stores/status.js";
	import { _ } 		  		from 'svelte-i18n'
	import { onMount } 			from "svelte";
	import {httpAPI} 			from "../../../lib/utils.js"
	import { serialQueue } 		from "../../../lib/queue.js";
	import {config_store} 		from "../../../lib/stores/config.js"
	import Box 					from "../../ui/Box.svelte"
	import Button 				from "../../ui/Button.svelte"
	import FirmwareUpdateModal 	from "./FirmwareUpdateModal.svelte"
	

	let restartOpenEvseState = ""
	let restartEspState = ""
	let resetEspState = ""
	let fw_modal_opened = false
	let fw_has_update = false
	let url = "https://api.github.com/repos/OpenEVSE/ESP32_WiFi_V4.x/releases/latest"
	let fw = {name: undefined, version: undefined, url: undefined}
	let alert_visible = false
	let alert2_visible = false
	let export_link
	let export_butn
	let import_file
	let import_butn

	onMount(()=>getFwUpdate())


	async function getFwUpdate() {
		if ($status_store.net_connected) {
			let fw_update_json = await httpAPI("GET",url)
			if (fw_update_json != "error") {
				fw.version = fw_update_json.name
				fw.name = $config_store.buildenv + ".bin"
				fw.html_url = fw_update_json.html_url
				let item = fw_update_json.assets.find(obj => {
					return obj.name === fw.name
				})
				if (item)
					fw.url = item.browser_download_url
				if (fw.version != $config_store.version) {
					fw_has_update = true
				}
			}
		}
	}
	
	async function restartOpenEvse() {
		restartOpenEvseState = "loading"
		const payload = "json=1&rapi=$FR"
		let res = await serialQueue.add(()=>httpAPI("POST","/r",payload, "text"))
		if (res == "set" )  {
			restartOpenEvseState = "ok"
			return true
		}
		else {
			// cheating as openEVSE api always answer HTTP 500 on reboot command
			restartOpenEvseState = "ok"
			return false
		}
	}

	async function restartESP() {
		restartEspState = "loading"
		let res = await serialQueue.add(()=>httpAPI("POST","/restart",null,"text"))
		if (res == "1" )  {
			restartEspState = "ok"
			return true
		}
		else {
			restartEspState = "error"
			return false
		}
	}

	async function resetESP(confirm=true) {
		if (confirm) {
			alert_visible = true
		}
		else {	
			resetEspState = "loading"
			alert_visible = false
			let res = httpAPI("GET", "/reset")
			if (res) {
				alert2_visible = true
				resetEspState = "ok"
			}
			else {
				alert_visible = true
				resetEspState = "error"
			}


		}
	}

	async function exportConfig() {
		export_butn = "loading"
		// get latest config
		if (!await config_store.download()) {
			export_butn = "error"
			return
		}
		// copy config_store
		let conf = {...$config_store}
		// remove all credentials with _DUMMY_PASSWORD from file
		Object.keys(conf).forEach(element => {
			if(
				conf[element] == "_DUMMY_PASSWORD"  	||
				element == "www_username" 				||
				element == "ssid" 						||
				element == "mqtt_supported_protocols" 	||
				element == "http_supported_protocols"	||
				element == "firmware"					||
				element == "protocol"					||
				element == "espflash"					||
				element == "espinfo"					||
				element == "buildenv"					||
				element == "version"					||
				element == "evse_serial"				||
				element == "wifi_serial"				
			
			) {
				delete conf[element]
			}
		});
		// create file
		const file = URL.createObjectURL(new Blob([JSON.stringify(conf,null,4)], { type: 'application/json' }))
		export_link.href =  file
		export_link.download = "config.json"
		export_butn = "ok"
		export_link.click()
		URL.revokeObjectURL(export_link.href);
	}

	async function importConfig() {
		import_butn = "loading"
		// let file = import_file.files[0]
		let reader = new FileReader();
 		reader.onload = async function() {
			let conf
			try {
				conf = JSON.parse(reader.result.toString())
			} catch (e) {
				console.log("Error parsing json")
				console.log(e)
				import_butn = "error"
				setTimeout(() => {
					import_file = null
				}, 2000);
				return false;
			}
			console.log(conf)
			let res
			if (await serialQueue.add(()=>config_store.upload(conf))) {
				import_butn = "ok"
			}
			else {
				console.log("Error uploading data")
				import_butn = "error"
			}
			setTimeout(() => {
					import_file = null
				}, 2000);
			if (import_butn == "error") {
				return false
			}
			else return true
			
		}
		reader.readAsText(import_file);
	}

</script>


<Box title={$_("config.titles.firmware")} icon="fa6-solid:microchip" back={true}>
	<table class="table is-fullwidth is-narrow has-text-dark">
		<thead>
			<tr class="has-background-info"	>
				<th class="has-text-white ">{$_("config.firmware.hardware")}</th>
				<th class="has-text-white">{$_("config.firmware.version")}</th>
				<th class="has-text-white has-text-centered" >{$_("config.firmware.action")}</th>
			</tr>
		</thead>
		<tbody>
			<tr>
				<td class="has-text-weight-bold">OpenEVSE</td>
				<td>{$config_store.firmware}</td>
				<td><div class="has-text-centered"><Button width="100px" size="is-responsive" name={$_("config.firmware.restart")} butn_submit={restartOpenEvse} state={restartOpenEvseState}/></div></td>
			</tr>
			<tr>
				<td class="has-text-weight-bold">OpenEVSE Wifi</td>
				<td>
					<div>{$config_store.version}</div>
					{#if fw.version && $config_store.version != fw.version}
					<div class="tag is-primary is-small has-text-weight-bold">
						{fw.version}
					</div>
					{:else if fw.version && $config_store.version == fw.version}
					<div class="tag is-info is-small has-text-weight-bold">
						{$_("config.firmware.up2date")}
					</div>
					{/if}
				</td>
				<td>
					<div class="has-text-centered is-flex is-flex-direction-column">
						<div class="mb-2">
							<Button  size="is-responsive" width="100px"name={$_("config.firmware.update")} butn_submit={()=>fw_modal_opened=true} color="{fw.version && $config_store.version != fw.version?"is-primary":"is-info"}" />
						</div>
						<div class="mb-2">
							<Button size="is-responsive" width="100px" name={$_("config.firmware.restart")} butn_submit={restartESP} state={restartEspState}/>
						</div>
						<div class="mb-2">
							<Button size="is-responsive" width="100px" name={$_("config.firmware.reset")} butn_submit={resetESP} state={resetEspState}/>
						</div>
					</div>
				</td>
			</tr>
		</tbody>
	</table>
	<div class="is-flex is-align-items-center is-justify-content-center px-6-tablet">
		<Borders grow>
			<div class="has-text-weight-bold has-text-dark mb-2">
				{$_("config.firmware.backup")}
			</div>
			<div class="mb-2">
				<div class="has-text-info has-text-weight-bold mb-1">
					{$_("config.firmware.backup-desc")}
				</div>
				<Button size="is-responsive" width="100px" state={export_butn} name={$_("config.firmware.export")} butn_submit={exportConfig}/>
				<div class="is-hidden">
					<a bind:this={export_link} href={null} >null</a>
				</div>
				
			</div>
			<div class=mb-2>
				{#if !import_file}
				<div>
					<SelectFile mini={true} width="100px" bind:file={import_file} ext=".json,.txt" title={$_("config.firmware.import")} />
				</div>

				{:else}
				<div>
					<Button size="is-responsive" color="is-primary" width="100px" state={import_butn} name={$_("config.firmware.upload")} butn_submit={importConfig} />
				</div>
				
				<div class="mt-2 is-inline-block">
					<RemovableTag
					action={()=> import_file = null}
					name={import_file.name}
				/>
				</div>
				{/if}
			</div>
		</Borders>
	</div>
	

</Box>
{#if fw_modal_opened}
<FirmwareUpdateModal bind:is_opened={fw_modal_opened} update={fw} />
{/if}
<AlertBox title={$_("warning")} body={$_("config.firmware.reset-warning")} label={$_("config.firmware.reset")} button={true} action={()=>resetESP(false)} bind:visible={alert_visible}></AlertBox>
<AlertBox title={$_("warning")}  body={$_("config.firmware.reset-reboot")} bind:visible={alert2_visible}/>