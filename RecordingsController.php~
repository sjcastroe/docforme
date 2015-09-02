<?php
define("LIDF_LEA_TABLE"			, "lidf_lea_config");

define("LICF_CCC_TABLE"			, "licf_ccc");
define("LICF_CDC_TABLE"			, "licf_cdc");
define("LICF_ACCOUNT_TABLE"		,"level1_access");

class RecordingsController extends AppController {

	public $uses = array('Recording','Cdr');
	public $layout = 'api';
	
	/**
	 * @api {post} ?object=recordings&action=read Read recordings
	 * @apiGroup	Recording
	 * @apiName		Read Recordings
	 * @apiHeader	{String}	Authorization Authorization value. Generally "Bearer ACCESSTOKEN"
	 * @apiParam	{String}	orig_callid		Origination Call-ID.
	 * @apiParam	{String}	term_callid		Termination Call-ID.
	 * @apiParam	{String}	callid			Termination Call-ID.
	 * @apiSampleRequest		?object=recording&action=read
	 * @apiPermission			User
	 * @apiSuccess	{String}	status			Status of the recording
	 * @apiSuccess	{String}	call_id			Call-ID associated with the recording
	 * @apiSuccess	{String}	time_open		Date & Time that the recording started
	 * @apiSuccess	{String}	time_close		Date & Time that the recording stopped
	 * @apiSuccess	{Number}	duration		Duration of the recording in seconds
	 * @apiSuccess	{Number}	size			Size of converted file in KByte
	 * @apiSuccess	{String}	time			Time reference for the URL
	 * @apiSuccess	{String}	geo_id 			Corresponding Orig Call ID, if call_id is for a Geo Tunneled entry 
	 * @apiSuccess	{String}	url				URL to Retrieve the Recording 
	 *
	 */
	 public function read($form) {

		$this->nslog("debug", "(".$this->name.".read) form " . print_r($form, true));
		
		
		if (isset($form['orig_callid']) || isset($form['term_callid']) || isset($form['callid'])) {
			
		}
		elseif (isset($form['aor']))
			return $this->read_lic($form);
		else
			$this->redirect(null, 400);

		$query = array();
		$query['object']	= "recording";
		
		if (isset($form['orig_callid']))
			$query['orig_callid'] = $form['orig_callid'];
		
		if (isset($form['term_callid']))
		{
			$query['term_callid'] = $form['term_callid'];
			$pGeoCallId = $this->__QueryGeoCallId($query['term_callid']);
			if ($pGeoCallId!=NULL)
				$query['geo_callid'] = $pGeoCallId['term_callid'];
		}

		if (isset($form['callid']))
			$query['callid'] = $form['callid'];
		
		if (isset($form['case_id']))
			$query['case_id'] = $form['case_id'];
				
		$this->nslog("debug", "(".$this->name.".read) query " . print_r($query, true));
		if (!Configure::read('NsSBusRecordingsInfo'))
		{
			$results = $this->__QueryLiCf($query);
		}
		else
		{
			$results['xml'] = $this->Recording->nsSendSbusReq($query, 'recordings', 'recording', "read");
		}
//		$this->nslog("debug", "(".$this->name.".read) result " . print_r($results, true));
		
		if ($results==NULL)
			$results['xml']['recording'] = array();
		elseif (false && isset($form['user']))
		{
			
			$this->nslog("debug", "(".$this->name.".read) result ". print_r($results, true));
			
			if(!isset($this->Device))
				$this->loadModel("Device");
			$q['conditions']['subscriber_domain']=$form['domain'];
			$q['conditions']['subscriber_name']=$form['user'];
			$q['fields'] = 'aor,subscriber_name,subscriber_domain';
			
			$posts = $this->Device->find('all',$q );
			
			for ($i = 0; $i < sizeof($posts); $i++) {
				$devices[] = $posts[$i]['Device']['aor'];
			}
			foreach ($results['xml']['recording'] as $idx =>$rec)
			{
				
				if (isset($devices))
					foreach ($devices as $aor)
					{
						if ($rec['case_id'] == $aor || "recording"==$rec['case_id'])
						{
							//$this->set('results', $results);
							//return;
						}
						else 
							unset($results['xml']['recording'][$idx]);
					}
					
				
					
			}
					
		}
		
//		$this->nslog("debug", "(".$this->name.".read) result before checking status\ngeo_id='". $pGeoCallId['orig_callid']."'\n" . print_r($results, true));
		if (!isset($results['xml']['recording']))
		{
			
		}
		else if (isset($results['xml']['recording'][0]))
		{
			foreach ($results['xml']['recording'] as $idx =>$pRecording)
			{
				if (!$this->__CheckRecordingStatus($results['xml']['recording'][$idx], $pGeoCallId))
				{
					unset($results['xml']['recording'][$idx]);
				}
			}
		}
		else
		{
			if (!$this->__CheckRecordingStatus($results['xml']['recording'], $pGeoCallId))
			{
				unset($results['xml']['recording']);
			}
		}
		$this->nslog("debug", "(".$this->name.".read) result " . print_r($results, true));
		$this->set('results', $results);
	}
	
	private function __CheckRecordingStatus(&$pRecording, $pGeoCallId)
	{
//		$this->nslog("debug", "(".$this->name.".read) geo_id='". $pGeoCallId['orig_callid']."'\npRecording " . print_r($pRecording, true));
		if (!isset($pRecording['status']))
			return false;
		else if ($pRecording['status']=="empty media")
			return false;
		else if ($pRecording['status']=="archive error")
			return false;
		else if (($pGeoCallId!=NULL) && ($pGeoCallId['term_callid'] == $pRecording['call_id']))
		{
			$pRecording['geo_id'] = $pGeoCallId['orig_callid'];
		}
		return true;
	}
	
	public function read_lic($form) {
	
		$this->nslog("debug", "(".$this->name.".read) form " . print_r($form, true));
	
		if (!isset($form['aor']) ) {
			$this->redirect(null, 400);
		}
		$results['xml']['lic'] = array();
		$pServers = $this->__QueryRecordingServers();
		foreach ($pServers as $pServer)
		{
			$query['lea_id'		]	=$pServer;
			$query['device_aor'	]	=$form['aor'];
			$query['case_id'	]	=$form['aor'];
			$query['valid_from'	]	='0000-00-00 00:00:00';
			$query['valid_to'	]	='0000-00-00 00:00:00';
			$query['ccc_intcpt'	]	='yes';
			$query['cdc_intcpt'	]	='yes';
			$query['object'		]	="lic";
				
			$lics   = $this->Lic->nsRead($query);
			$this->nslog("debug", "(".$this->name.".read) lics " . print_r($lics, true));
			if(isset($lics['xml']['lic']))
				foreach ($lics['xml']['lic'] as $val)
					$results['xml']['lic'][] =$val;		
			
		}
		$this->nslog("debug", "(".$this->name.".read) results " . print_r($results, true));
		$this->set('results', $results);
	}

	public function create($form) {

		$this->nslog("debug", "(".$this->name.".create) form " . print_r($form, true));
		
		if (isset($form['aor'])) {
		}
		else {
			return $this->errorResponse(400, 'Please submit "aor" parameter');
		}
		
		$this->loadModel('Lic');
		$pServers = $this->__QueryRecordingServers();
		foreach ($pServers as $pServer)
		{
			$query['lea_id'		]	=$pServer;
			$query['device_aor'	]	=$form['aor'];
			$query['case_id'	]	=$form['aor'];
			$query['valid_from'	]	='0000-00-00 00:00:00';
			$query['valid_to'	]	='0000-00-00 00:00:00';
			$query['ccc_intcpt'	]	='yes';
			$query['cdc_intcpt'	]	='yes';
			
			
			if (!$this->Lic->nsCreate($query,  'lidf_events', 'lic'))
				$this->errorResponse(400, 'Fail to create LIC');
		}
		
		$this->redirect(null, 200);
	}
	
	public function update($form) {
	
		$this->nslog("debug", "(".$this->name.".update) form " . print_r($form, true));
	
		if (isset($form['aor'])) {
		}
		else {
			return $this->errorResponse(400, 'Please submit "aor" parameter');
		}
	
		$this->loadModel('Lic');
		$pServers = $this->__QueryRecordingServers();
		foreach ($pServers as $pServer)
		{
			$query['lea_id'		]	=$pServer;
			$query['device_aor'	]	=$form['aor'];
			$query['case_id'	]	=$form['aor'];
			$query['valid_from'	]	='0000-00-00 00:00:00';
			$query['valid_to'	]	='0000-00-00 00:00:00';
			$query['ccc_intcpt'	]	='yes';
			$query['cdc_intcpt'	]	='yes';
				
				
			if (!$this->Lic->nsUpdate($query,  'lidf_events', 'lic'))
				$this->errorResponse(400, 'Fail to update LIC');
		}
	
		$this->redirect(null, 200);
	}
	
	public function delete($form) {

		$this->nslog("debug", "(".$this->name.".delete) form " . print_r($form, true));
		
		if (isset($form['aor'])) {
		}
		else {
			return $this->errorResponse(400, 'Please submit "aor" parameter');
		}
		
		$this->loadModel('Lic');
		$pServers = $this->__QueryRecordingServers();
		foreach ($pServers as $pServer)
		{
			$query['lea_id'		]	=$pServer;
			$query['device_aor'	]	=$form['aor'];
			$query['case_id'	]	=$form['aor'];
			
			if (!$this->Lic->nsDelete($query,  'lidf_events', 'lic'))
				$this->errorResponse(400, 'Fail to delete LIC');
		}
		
		$this->redirect(null, 200);
	}
	
	private function __GetLocalHostname()
	{
		return trim(shell_exec("hostname"));
	}
	
	
	public function eventCreate($event) {
	
		$this->nslog("debug", "(".$this->name.".eventCreate) event " . print_r($event, true));
	
		if (!isset($event['object'])) {
		}
		else if ($event['object']=='lic') {
			if (!isset($this->Lic))
				$this->loadModel('Lic');
			$this->Lic->nsCreateFromSbus($event);
		}
	}
	
	public function eventDelete($event) {
	
		$this->nslog("debug", "(".$this->name.".eventDelete) event " . print_r($event, true));
	
		if (!isset($event['object'])) {
		}
		else if ($event['object']=='lic') {
			if (!isset($this->Lic))
				$this->loadModel('Lic');
			$this->Lic->nsDeleteFromSbus($event);
		}
	}
	

	private function _GetPlaybackUrl($strAccElm, $strCaseId, $strCallId,  $strCccId, $strUid, $strTime, $strPwd)
	{
		$strMd5Pwd	= md5($strPwd);
		$strTime	= gmdate("YmdHis", strtotime($strTime)); 

		$strDbHost = $this->Recording->getDataSource()->config['host'];
		if ($strDbHost =="localhost")
			$server = $_SERVER["SERVER_NAME"];
		else
			$server = $strDbHost;
		
		$strUrl = "https://".$server."/LiCf/playback.php";
		$strUrl.="?submit="	."PLAY";
		$strUrl.="&accel="	.$strAccElm;
		$strUrl.="&casid="	.$strCaseId;
		$strUrl.="&cccid="	.$strCccId;
		$strUrl.="&calid="	.$strCallId;
		$strUrl.="&uid="	.$strUid;
		$strUrl.="&time="	.$strTime;
		$strUrl.="&auth="	.md5($strAccElm.$strCaseId.$strCccId.$strCallId.$strUid.$strPwd.$strTime);
		
	 	return $strUrl;
	}
	
	private function __QueryRecordingCredential($uid) {
		$sql = "SELECT";
		$sql.=" password_md5 as password";
		$sql.=" FROM ".LICF_ACCOUNT_TABLE;
		$sql.=" WHERE";
		$sql.=" login='"	.$this->NsStringEscape($uid)		."'";
		$sql.=" AND";
		$sql.=" status='"	."active"							."'";
		$sql.=" LIMIT 0,1";
	
		$pRows = $this->Recording->query($sql);
//		$this->nslog("debug", "(".$this->name.".__QueryRecordingCredential) uid='".$uid."' pRows " . print_r($pRows, true));
		
		if (isset($pRows[0]) && isset($pRows[0][LICF_ACCOUNT_TABLE])  && isset($pRows[0][LICF_ACCOUNT_TABLE]["password"]))
			return $pRows[0][LICF_ACCOUNT_TABLE]["password"];
		else
			return NULL;
	}
	
	private function __QueryGeoCallId($strTermCallId)
	{
		$geoCallId = array();
		$query['conditions']['orig_callid']	=$strTermCallId;
		$query['conditions']['release_code']="end";
		$query['conditions']['by_action']	="GeoTunnel";

//		$this->nslog("debug", "(".$this->name.".__QueryGeoCallId) query " . print_r($query, true));
		$results = $this->Cdr->find('first', $query);
//		$this->nslog("debug", "(".$this->name.".__QueryGeoCallId) results " . print_r($results, true));
		
		if (!isset($results['Cdr']['term_callid']))
			return NULL;
		else
		{
			$pGeoCallId = array();
			$pGeoCallId['term_callid'] = $results['Cdr']['term_callid'];
			$pGeoCallId['orig_callid'] = $results['Cdr']['orig_callid'];
			return $pGeoCallId;
		}
	}
	
	private function __QueryRecordingInfo($pQuery, $bFromSBus = false) {
//		$this->nslog("debug", "(".$this->name.".__QueryRecordingInfo) pQuery " . print_r($pQuery, true));

		$geoLinks = array();
		
		$strOR = "";
		$strAND = "";
		
		if (isset($pQuery['orig_callid']))
			$strOrigCallId = $this->NsStringEscape($pQuery['orig_callid']);
		else
			$strOrigCallId = "";
		
		if (isset($pQuery['geo_callid']))
			$strGeoOrigCallId = $this->NsStringEscape($pQuery['geo_callid']);
		else
			$strGeoOrigCallId = "";
				
		if (isset($pQuery['term_callid']))
		{
			$strTermCallId = $this->NsStringEscape($pQuery['term_callid']);
		}
		else
			$strTermCallId = "";
		
		if (isset($pQuery['callid']))
			$strCallId = $this->NsStringEscape($pQuery['callid']);
		else
			$strCallId = "";
		
		if (isset($pQuery['case_id']))
			$strCaseId = $this->NsStringEscape($pQuery['case_id']);
		else
			$strCaseId = "";
		
		$sql = "SELECT";
		$sql.=" ".LICF_CCC_TABLE.".conversion_status as status";
		$sql.=",".LICF_CDC_TABLE.".acc_elm";
		$sql.=",".LICF_CDC_TABLE.".case_id";
		$sql.=",".LICF_CDC_TABLE.".call_id";
		$sql.=",".LICF_CDC_TABLE.".ccc_id";
		$sql.=",".LICF_CCC_TABLE.".*";
		$sql.=","."UTC_TIMESTAMP() as time";
		$sql.=" FROM ".LICF_CDC_TABLE;
		$sql.=" LEFT JOIN ".LICF_CCC_TABLE;
		$sql.=" ON";
		$sql.=" ".LICF_CDC_TABLE.".ccc_id"	."="	.LICF_CCC_TABLE.".ccc_id";
		$sql.=" AND";
		$sql.=" ".LICF_CDC_TABLE.".call_id"	."="	.LICF_CCC_TABLE.".call_id";
		$sql.=" WHERE";
		if ($strCaseId != "") {
			$sql.=" ";
			$sql.=$strOR.LICF_CDC_TABLE.".case_id"	."='"	.$this->NsStringEscape($strCaseId)		."'";
			$strAND = " AND ";
		}
		$sql.=$strAND." (";
		if ($strCallId != "") {
			$sql.=$strOR.LICF_CDC_TABLE.".call_id"	."='"	.$this->NsStringEscape($strCallId)		."'";
			$strOR = " OR ";
		}
		if ($strOrigCallId != "") {
			$sql.=$strOR.LICF_CDC_TABLE.".call_id"	."='"	.$this->NsStringEscape($strOrigCallId)		."'";
			$strOR = " OR ";
		}
		if ($strTermCallId != "") {
			$sql.=$strOR.LICF_CDC_TABLE.".call_id"	."='"	.$this->NsStringEscape($strTermCallId)		."'";
			$strOR = " OR ";
		}
		if ($strGeoOrigCallId != "") {
			$sql.=$strOR.LICF_CDC_TABLE.".call_id"	."='"	.$this->NsStringEscape($strGeoOrigCallId)		."'";
			$strOR = " OR ";
		}
		
		$sql.=" )";
		$sql.=" ORDER BY time_open ASC";
//		$sql.=" LIMIT 0,1";
	
		$pRows = $this->Recording->query($sql);
//		$this->nslog("debug", "(".$this->name.".__QueryRecordingInfo) pRows " . print_r($pRows, true));
		
		$results = array();
		foreach ($pRows as $pRow)
		{
			$result = array(); 
			foreach ($pRow as $pTable)
			{
				foreach ($pTable as $key => $value)
				{
					$result[$key] = $value;
				}
			}
			$result['duration'] = 0;
			if(isset($result["time_close"]) && isset($result["time_open"]))
			{
				$result['duration'	]=strtotime($result["time_close"]) - strtotime($result["time_open"]);

				if (isset($result['acc_elm']))
					$result['url'		]=$this->_GetPlaybackUrl($result['acc_elm'], $result['case_id'], $result['call_id'],  $result['ccc_id'], $pQuery['uid'], $result['time'], $pQuery['pwd']);

				if (isset($geoLinks[$result['call_id']]))
					$result["geo_id"] = $geoLinks[$result['call_id']];
				else
					$result["geo_id"] = "na";
			}
			
			unset($result["type"]);
			unset($result["orig_aor"]);
			unset($result["term_aor"]);
			unset($result["redir_from"]);
			unset($result["redir_to"]);
			unset($result["timeLastPacket"]);
			unset($result["archiveStatus"]); 
			unset($result["file_0"]);
			unset($result["file_1"]);
			unset($result["file_2"]);
			unset($result["cnt_0"]);
			unset($result["cnt_1"]);
			unset($result["acc_elm"]);
			unset($result["case_id"]);
			unset($result["ccc_id"]);
				
//			$this->nslog("debug", "(".$this->name.".__QueryRecordingInfo) result " . print_r($result, true));
			
			if (($result['duration']>0) && (!isset($result['size_2']) || ($result['size_2']>0)))
			{
				$result['size'] = $result['size_2'];
				unset($result['size_2']);
				unset($result["conversion_status"]);
				$results[] = $result;
			}
			elseif (isset($result["size_2"]) && $result["size_2"]=="0" && $result["conversion_status"] !="empty"  && $result["conversion_status"] !="deleted")
			{
				$result['size'] = "0";
				unset($result["conversion_status"]);
				unset($result['size_2']);
				$results[] = $result;
			}
		}
		
		return $results;
	}
	
	private function __QueryLiCf($pQuery) {
		$results['xml'] = array();
		
		$pQuery["uid"]	= $this->__GetLocalHostname();
		$pQuery["pwd"]	= $this->__QueryRecordingCredential	($pQuery["uid"]);
		if ($pQuery["pwd"] == NULL)
			return $results;
		
		$results['xml']['recording'] = $this->__QueryRecordingInfo		($pQuery);
		
		if (count($results['xml']['recording']) < 1)
			$results['xml'] = array();
		
//		$this->nslog("debug", "(".$this->name.".__QueryLiCf) results " . print_r($results, true));
		
		return $results;
	}
	
	private function __QueryRecordingServers() {
		$pRows = array();
		
		$sql = "SELECT";
		$sql.=" lea_id";
		$sql.=" FROM ".LIDF_LEA_TABLE;
		$sql.=" WHERE";
		$sql.=" lea_id LIKE 'Recording-Server%'";
		$sql.=" ORDER BY lea_id ASC";
	
		if (!isset($this->Lic))
			$this->loadModel('Lic');
		$pRows = $this->Lic->query($sql);
		
		$results = array();
		foreach ($pRows as $pRow)
		{
			$results[] = $pRow[LIDF_LEA_TABLE]['lea_id'];
		}
		
		return $results;
	}
	
	public function eventRead($event) {
		if (!isset($event['orig_callid']) && !isset($event['term_callid']) && !isset($event['callid'])) {
			$this->redirect(null, 400);
		}
		
//		$this->nslog("debug", "(".$this->name.".eventRead) event " . print_r($event, true));
		$results = $this->__QueryLiCf($event);
//		$this->nslog("debug", "(".$this->name.".eventRead) result " . print_r($results, true));
		
		if (isset($results['xml']) && isset($results['xml']['recording']))
			$this->responseWithXml($results);
		else
			$this->errorResponse(404, "Not Found");
	}
	
}
?>
