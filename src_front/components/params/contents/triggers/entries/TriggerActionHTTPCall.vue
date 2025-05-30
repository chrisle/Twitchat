<template>
	<div class="triggeractionhttpcall triggerActionForm">
		<ParamItem :paramData="param_url" v-model="action.url"
			:error="securityError"
			:errorMessage="$t('triggers.actions.http_ws.protocol_error')" />

		<ParamItem class="row item" :paramData="param_method" v-model="action.method">
			<ParamItem class="row child item" :paramData="param_sendAsBody" v-model="action.sendAsBody" v-if="action.method == 'POST' || action.method == 'PATCH'" noBackground />
			<ParamItem class="row child item" :paramData="param_custom_body" v-model="action.customBody" v-if="action.method == 'POST' || action.method == 'PATCH'" noBackground />
		</ParamItem>

		<ParamItem class="row item" :paramData="param_customHeaders" v-model="action.customHeaders">
			<div class="headerList">
				<div class="header head" v-if="action.headers && action.headers.length > 0">
					<div>Key</div>
					<div>Value</div>
				</div>
				<div v-for="(a, index) in action.headers" class="header">
					<input type="text" v-model="action.headers![index].key">
					<ParamItem :paramData="getParamHeaderValue(index)" placeholdersAsPopout v-model="action.headers![index].value" noBackground />
					<TTButton class="deleteBt" icon="trash" @click="delHeader(index)" alert />
				</div>
				<TTButton class="addBt" icon="add" v-if="(action.headers || []).length < 20" @click="addHeader()">{{ $t("triggers.actions.http_ws.add_headerBt") }}</TTButton>
			</div>
		</ParamItem>

		<div class="card-item tags">
			<p class="title" v-if="parameters.length > 0">{{ $t("triggers.actions.http_ws.select_param") }}</p>

			<div class="params">
				<ParamItem class="toggleAll" noBackground :paramData="param_toggleAll" v-model="param_toggleAll.value" @click.native="toggleAll()" v-if="parameters.length > 3" />

				<div class="card-item" v-for="p in parameters" :key="p.placeholder.tag" @click="p.enabled = !p.enabled; onToggleParam()">
					<div class="taginfo">
						<div class="tag"><mark>{{ p.placeholder.tag }}</mark></div>
						<span>{{ $t(p.placeholder.descKey, p.placeholder.descReplacedValues || {}) }}</span>
					</div>
					<ToggleButton v-model="p.enabled" @change="onToggleParam()" />
				</div>
			</div>
		</div>

		<TriggerActionHttpPlaceholder :placeholderList="action.outputPlaceholderList" />
	</div>
</template>

<script lang="ts">
import ToggleButton from '@/components/ToggleButton.vue';
import ParamItem from '@/components/params/ParamItem.vue';
import type { ITriggerPlaceholder, TriggerActionHTTPCallData, TriggerActionHTTPCallDataAction, TriggerData } from '@/types/TriggerActionDataTypes';
import type { TwitchatDataTypes } from '@/types/TwitchatDataTypes';
import { watch } from 'vue';
import {toNative,  Component, Prop } from 'vue-facing-decorator';
import AbstractTriggerActionEntry from './AbstractTriggerActionEntry';
import TTButton from '@/components/TTButton.vue';
import TriggerActionHttpPlaceholder from './common/TriggerActionHttpPlaceholder.vue';

@Component({
	components:{
		TTButton,
		ParamItem,
		ToggleButton,
		TriggerActionHttpPlaceholder,
	},
	emits:["update"]
})
class TriggerActionHTTPCall extends AbstractTriggerActionEntry {

	@Prop
	declare action:TriggerActionHTTPCallData;

	@Prop
	declare triggerData:TriggerData;

	public securityError:boolean = false;
	public placeholderList:ITriggerPlaceholder<any>[] = [];
	public parameters:{placeholder:ITriggerPlaceholder<any>, enabled:boolean}[] = [];
	public param_url:TwitchatDataTypes.ParameterData<string> = {type:"string", value:"", placeholder:"https://...", labelKey:"triggers.actions.http_ws.url"};
	public param_method:TwitchatDataTypes.ParameterData<TriggerActionHTTPCallDataAction, TriggerActionHTTPCallDataAction> = {type:"list", value:"GET", listValues:[], labelKey:"triggers.actions.http_ws.method"};
	public param_outputPlaceholder:TwitchatDataTypes.ParameterData<string> = {type:"string", value:"", labelKey:"triggers.actions.http_ws.output_placeholder", maxLength:30, allowedCharsRegex:"A-z0-9_"};
	public param_custom_body:TwitchatDataTypes.ParameterData<string> = {type:"string", value:"", longText:true, labelKey:"triggers.actions.http_ws.custom_body", placeholderKey:"triggers.actions.http_ws.custom_body_placeholder", maxLength:5000};
	public param_toggleAll:TwitchatDataTypes.ParameterData<boolean> = {type:"boolean", value:false, labelKey:"chat.filters.select_all" };
	public param_sendAsBody:TwitchatDataTypes.ParameterData<boolean> = {type:"boolean", value:true, labelKey:"triggers.actions.http_ws.send_as_body" };
	public param_customHeaders:TwitchatDataTypes.ParameterData<boolean> = {type:"boolean", value:false, labelKey:"triggers.actions.http_ws.custom_headers" };
	public param_headerValues:TwitchatDataTypes.ParameterData<string>[] = [];

	public getParamHeaderValue(index:number):TwitchatDataTypes.ParameterData<string> {
		if(this.param_headerValues.length>index) return this.param_headerValues[index];
		const param:TwitchatDataTypes.ParameterData<string> = {type:'string', value:"", placeholderList:this.placeholderList};
		this.param_headerValues[index] = param;
		return param;
	}

	public beforeMount():void {
		if(!this.action.outputPlaceholderList) this.action.outputPlaceholderList = [];
		this.param_method.listValues	= (["GET","PUT","POST","PATCH","DELETE"] as TriggerActionHTTPCallDataAction[]).map(v=> {return {value:v, label:v}});

		watch(()=>this.action.url, ()=> {
			const url = this.action.url;
		    // This regular expression whitelists http or https on reserved private 
    		// IPv4 addresses (Class A: 10.0.0.0/8, Class B: 172.16.0.0/12, 
    		// Class C: 192.168.0.0/16), localhost, with or without a port number.
    		// Reference: https://en.wikipedia.org/wiki/Private_network
    		const whitelistRegex = /^(https?:\/\/)?(localhost|127\.0\.0\.1|10\.\d+\.\d+\.\d+|172\.(1[6-9]|2\d|3[0-1])\.\d+\.\d+|192\.168\.\d+\.\d+)(:\d+)?(\/.*)?$/i;
			this.securityError = !whitelistRegex.test(url);
		});

		this.onToggleParam();
	}

	public onToggleParam():void {
		const params:string[] = this.parameters.filter(v=>v.enabled).map(v=> v.placeholder.tag);
		this.action.queryParams = params;

		let allSelected = true;
		this.parameters.forEach(v=> allSelected &&= v.enabled);
		this.param_toggleAll.value = allSelected
	}

	/**
	 * Called when clicking "all" toggle
	 */
	public toggleAll():void {
		//Leave it time for the model value to update.
		//Not usign @change because the value is changed on onToggleParam() based
		//on currently enabled parameters which would conflict with this handler
		//if it was using @change.
		window.setTimeout(() => {
			this.parameters.forEach(v=> v.enabled = this.param_toggleAll.value);
			this.onToggleParam();
		}, 0);
	}

	/**
	 * Called when the available placeholder list is updated
	 */
	public onPlaceholderUpdate(list:ITriggerPlaceholder<any>[]):void {
		this.placeholderList = list;
		this.parameters = list.map(v=> {
			return  {
				placeholder:v,
				enabled:!this.action.queryParams || this.action.queryParams.includes(v.tag),
			}
		});

		this.param_url.placeholderList = list;
		this.param_custom_body.placeholderList = list;
	}

	/**
	 * Called when "add header" button is clicked
	 */
	public addHeader():void {
		if(!this.action.headers) this.action.headers = [];
		this.action.headers.push({
			key:"",
			value:"",
		});
	}

	/**
	 * Called when "delete header" button is clicked
	 */
	public delHeader(index:number):void {
		this.action.headers?.splice(index, 1);
	}

}
export default toNative(TriggerActionHTTPCall);
</script>

<style scoped lang="less">
.triggeractionhttpcall{

	.tags {
		.title {
			margin-bottom: .5em;
		}
	}

	.params {
		gap: 2px;
		display: flex;
		flex-direction: column;
		font-size: .8em;
		max-height: 375px;
		overflow-y: auto;
		.card-item {
			display: flex;
			flex-direction: row;
			align-items: center;
			cursor: pointer;
			flex-shrink: 0;
			&:hover {
				background-color: var(--color-light-fade);
			}
			.taginfo {
				gap: .5em;
				flex-grow: 1;
				display: flex;
				flex-direction: column;

				.tag {
					word-break: break-all;
				}
			}
		}
		.toggleAll {
			justify-self: flex-end;
			align-self: flex-end;
			margin-bottom: 2px;
			// margin-right: 2.25em;
			width: fit-content;
			margin-right: .5em;
			margin-bottom: .5em;
		}
	}
	.headerList {
		gap: .5em;
		display: flex;
		flex-direction: column;
		.header {
			gap: .5em;
			display: flex;
			flex-direction: row;
			flex: 1;
			&.head {
				gap: 0;
				width: calc(100% - 1.6em);
				justify-content: space-around;
				div {
					background-color: var(--grayout);
					padding: .25em;
					text-align: center;
					&:first-child{
						margin-right: 1px;
						border-top-left-radius: var(--border-radius);
						border-bottom-left-radius: var(--border-radius);
					}
					&:not(:first-child){
						border-top-right-radius: var(--border-radius);
						border-bottom-right-radius: var(--border-radius);
					}
				}
			}
			*:not(.button) {
				width: 50%;
				flex-grow: 1;
				// width: 100%;
				// min-width: unset;
				// max-width: unset;
			}
			.deleteBt {
				flex-shrink: 0;
			}
		}
		.addBt {
			align-self: center;
		}
	}
}
</style>
