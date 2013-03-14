obiwon
======

mobile platform intended for oracle financials.

<?php

class API extends Handler {

  const API_LEVEL  = 4;

	const STATUS_OK  = 0;
	const STATUS_ERR = 1;

	private $seq;

	function before($method) {
		if (parent::before($method)) {
			header("Content-Type: text/json");

			if (!$_SESSION["uid"] && $method != "login" && $method != "isloggedin") {
				print $this->wrap(self::STATUS_ERR, array("error" => 'NOT_LOGGED_IN'));
				return false;
			}

			if ($_SESSION["uid"] && $method != "logout" && !get_pref($this->link, 'ENABLE_API_ACCESS')) {
				print $this->wrap(self::STATUS_ERR, array("error" => 'API_DISABLED'));
				return false;
			}

			$this->seq = (int) $_REQUEST['seq'];

			return true;
		}
		return false;
	}

	function wrap($status, $reply) {
		print json_encode(array("seq" => $this->seq,
			"status" => $status,
			"content" => $reply));
	}

	function getVersion() {
		$rv = array("version" => VERSION);
		print $this->wrap(self::STATUS_OK, $rv);
	}

	function getApiLevel() {
		$rv = array("level" => self::API_LEVEL);
		print $this->wrap(self::STATUS_OK, $rv);
	}

	function login() {
		$login = db_escape_string($_REQUEST["user"]);
		$password = $_REQUEST["password"];
		$password_base64 = base64_decode($_REQUEST["password"]);

		if (SINGLE_USER_MODE) $login = "admin";

		$result = db_query($this->link, "SELECT id FROM ttrss_users WHERE login = '$login'");

		if (db_num_rows($result) != 0) {
			$uid = db_fetch_result($result, 0, "id");
		} else {
			$uid = 0;
		}

		if (!$uid) {
			print $this->wrap(self::STATUS_ERR, array("error" => "LOGIN_ERROR"));
			return;
		}

		if (get_pref($this->link, "ENABLE_API_ACCESS", $uid)) {
			if (authenticate_user($this->link, $login, $password)) {               // try login with normal password
				print $this->wrap(self::STATUS_OK, array("session_id" => session_id(),
					"api_level" => self::API_LEVEL));
			} else if (authenticate_user($this->link, $login, $password_base64)) { // else try with base64_decoded password
				print $this->wrap(self::STATUS_OK,	array("session_id" => session_id(),
					"api_level" => self::API_LEVEL));
			} else {                                                         // else we are not logged in
				print $this->wrap(self::STATUS_ERR, array("error" => "LOGIN_ERROR"));
			}
		} else {
			print $this->wrap(self::STATUS_ERR, array("error" => "API_DISABLED"));
		}
