# Python ldap.LDAPError() Examples

The following are _42_ code examples for showing how to use
_ldap.LDAPError()_. They are extracted from open source Python projects. You
can vote up the examples you like or vote down the exmaples you don't like.
You can also save this page to your account.

Example 1

    
    
    def auth(self, username, password): 
      """ Authenticate a user against the ldap server """ 
      dn = 'uid={0},{1}'.format(username, self.base_dn) 
      try: 
        with self._ldap_connection() as ldap_cxn: 
          ldap_cxn.simple_bind_s(dn, password) 
            return True 
      except ldap.INVALID_CREDENTIALS: 
        raise except ldap.INVALID_DN_SYNTAX: 
        self.bus.log('Invalid DN syntax in configuration: {0}'.format(self.base_dn), 40) 
        raise 
       except ldap.LDAPError as e: self.bus.log('LDAP Error: {0}'.format(e.message['desc'] if 'desc' in e.message else str(e)), level=40, traceback=True) raise 

Example 2

    
    
    def get_user_by_uid(self, uid): 
    """ Lookup an ldap user by uid and return a dict with all known and anonymously accessible attributes. """ 
      searchfilter = '(uid={0})'.format(uid) 
      try: 
        user = self._search(searchfilter) 
        if not user or len(user) > 1: 
          return {} 
        else: return user[0][1] 
      except ldap.LDAPError as e: 
        self.bus.log('LDAP Error: {0}'.format(e.message['desc'] 
        if 'desc' in e.message else str(e)), level=40, traceback=True) raise 

Example 3

    
    
    def get_user_by_email(self, email_address): 
    """ Lookup an ldap user by email address and return a dict with all known and anonymously accessible attributes. """ 
      searchfilter = '(mail={0})'.format(email_address) 
      try: 
        user = self._search(searchfilter) 
        if not user or len(user) > 1: 
          return {} 
        else: 
          return user[0][1] 
      except ldap.LDAPError as e:
        self.bus.log('LDAP Error: {0}'.format(e.message['desc'] if 'desc' in e.message else str(e)), level=40, traceback=True) raise 

Example 4

    
    
    def delete_sshpubkey(self, username, sshpubkey): 
    """ Add an sshPublicKey attribute to the user's dn """ 
    dn = 'uid={0},{1}'.format(username, self.base_dn) 
    try: with self._ldap_connection() as ldap_cxn: 
      ldap_cxn.simple_bind_s(self.bind_dn, self.bind_pw) 
      mod_list = [(ldap.MOD_DELETE, 'sshPublicKey', str(sshpubkey))] ldap_cxn.modify_s(dn, mod_list) 
    except (ldap.INVALID_CREDENTIALS, ldap.INSUFFICIENT_ACCESS, ldap.LDAPError) as e: 
      self.bus.log('LDAP Error: {0}'.format(e.message['desc'] if 'desc' in e.message else str(e)), level=40, traceback=True) 
      raise 

Example 5

    
    
    def __init__(self, uri, cn, dc, secret): self.conn = None self.uri = uri self.dc = dc self.secret = secret try: self.conn = ldap.initialize(self.uri) self.conn.protocol_version = ldap.VERSION3 self.conn.simple_bind_s(cn+","+self.dc,self.secret) print("Connection established.") except ldap.INVALID_CREDENTIALS: print("Your username or password is incorrect.") sys.exit() except ldap.LDAPError as e: if type(e.message) == dict and e.message.has_key('desc'): print(e.message['desc']) else: print(e) sys.exit() 

Example 6

    
    
    def getusercredbyuid(self, username): baseDN = "ou=People,"+self.dc searchScope = ldap.SCOPE_SUBTREE searchFilter = "uid="+username searchAttrList = ["givenName","sn"] try: # Result is a list of touple (one for each match) of dn and attrs. # Attrs is a dict, each value is a list of string. result = self.conn.search_s(baseDN, searchScope, searchFilter, searchAttrList) if ( result==[] ): print("User %s does not exist!", username) return result = [value[0] for value in result[0][1].values()] # Returns 2-element list, name and surname return result except ldap.LDAPError as e: print(e) return 

Example 7

    
    
    def get_real_name(self): """ Attempt to retrieve the LDAP Common Name of the given login name. """ encoding = _config.get('ldap', 'encoding') user_dn = self.get_user_dn().encode(encoding) name_attr = _config.get('ldap', 'name_attr') try: res = self.ldap.search_s(user_dn, ldap.SCOPE_BASE, '(objectClass=*)', [name_attr]) except ldap.LDAPError: _logger.exception("Caught exception while retrieving user name " "from LDAP, returning None as name") return None # Just look at the first result record, since we are searching for # a specific user record = res[0][1] name = record[name_attr][0] return name 

Example 8

    
    
    def __init__(self,ldap_host=None,base_dn=None,user=None,password=None): if not ldap_host: ldap_host = LDAP_HOST if not base_dn: self.base_dn = BASE_DN if not user: user = USER if not password: password = PASSWORD try: self.ldapconn = ldap.open(ldap_host) self.ldapconn.simple_bind(user,password) except ldap.LDAPError,e: print e #?????????????????dn,??dn??????????????
    #?ldap???cn=username,ou=users,dc=gccmx,dc=cn,??????????????DN 

Example 9

    
    
    def ldap_search_dn(self,uid=None): obj = self.ldapconn obj.protocal_version = ldap.VERSION3 searchScope = ldap.SCOPE_SUBTREE retrieveAttributes = None searchFilter = "cn=" + uid try: ldap_result_id = obj.search(self.base_dn, searchScope, searchFilter, retrieveAttributes) result_type, result_data = obj.result(ldap_result_id, 0)
    #??????
    #('cn=django,ou=users,dc=gccmx,dc=cn',
    # { 'objectClass': ['inetOrgPerson', 'top'],
    # 'userPassword': ['{MD5}lueSGJZetyySpUndWjMBEg=='],
    # 'cn': ['django'], 'sn': ['django'] } )
    # if result_type == ldap.RES_SEARCH_ENTRY: #dn = result[0][0] return result_data[0][0] else: return None except ldap.LDAPError, e: print e #?????????????? 

Example 10

    
    
    def ldap_get_user(self,uid=None): obj = self.ldapconn obj.protocal_version = ldap.VERSION3 searchScope = ldap.SCOPE_SUBTREE retrieveAttributes = None searchFilter = "cn=" + uid try: ldap_result_id = obj.search(self.base_dn, searchScope, searchFilter, retrieveAttributes) result_type, result_data = obj.result(ldap_result_id, 0) if result_type == ldap.RES_SEARCH_ENTRY: username = result_data[0][1]['cn'][0] email = result_data[0][1]['mail'][0] nick = result_data[0][1]['sn'][0] result = {'username':username,'email':email,'nick':nick} return result else: return None except ldap.LDAPError, e: print e #????????????????????LDAP???boolean? 

Example 11

    
    
    def authenticate(username, password): records = get_user_records(username) dila_permission = check_group_membership(username) if records and dila_permission: user_dn, user_attributes = records[0] with initialize_connection() as connection: try: connection.simple_bind_s(user_dn, password) except ldap.LDAPError: return ANONYMOUS_USER else: encoding = config.LDAP_ENCODING first_name = user_attributes.get(config.LDAP_USER_ATTRIBUTE_MAP['first_name'])[0].decode(encoding) last_name = user_attributes.get(config.LDAP_USER_ATTRIBUTE_MAP['last_name'])[0].decode(encoding) is_superuser = check_group_membership(username, config.LDAP_SUPERUSER_GROUP_CN) return structures.User( authenticated=True, username=username, first_name=first_name, last_name=last_name, is_superuser=is_superuser ) else: return ANONYMOUS_USER 

Example 12

    
    
    def ldap_initialize(module, server): ldapmodule_trace_level = 1 ldapmodule_trace_file = sys.stderr ldap._trace_level = ldapmodule_trace_level try: conn = ldap.initialize( server, trace_level=ldapmodule_trace_level, trace_file=ldapmodule_trace_file ) except ldap.LDAPError as e: fail_msg = "LDAP Error initializing: {}".format(ldap_errors(e)) module.fail_json(msg=fail_msg) return conn 

Example 13

    
    
    def ldap_bind_with_user(module, conn, username, password): result = False try: conn.simple_bind_s(username, password) result = True except ldap.INVALID_CREDENTIALS: fail_msg = "Invalid Credentials for user {}".format(username) module.fail_json(msg=fail_msg) except ldap.LDAPError as e: fail_msg = "LDAP Error Binding user: {}: ERROR: {}".format(username, ldap_errors(e)) module.fail_json(msg=fail_msg) return result 

Example 14

    
    
    def getDefaultNamingContext(self): try: newCon = ldap.initialize('ldap://{}'.format(self.dc_ip)) newCon.simple_bind_s('','') res = newCon.search_s("", ldap.SCOPE_BASE, '(objectClass=*)') rootDSE = res[0][1] except ldap.LDAPError, e: print "[!] Error retrieving the root DSE" print "[!] {}".format(e) sys.exit(1) if not rootDSE.has_key('defaultNamingContext'): print "[!] No defaultNamingContext found!" sys.exit(1) defaultNamingContext = rootDSE['defaultNamingContext'][0] self.domainBase = defaultNamingContext newCon.unbind() return defaultNamingContext 

Example 15

    
    
    def exact(self): try: results = self.connection.search_s( self.dn, ldap.SCOPE_BASE, attrlist=[self.name]) except ldap.LDAPError: e = get_exception() self.module.fail_json( msg="Cannot search for attribute %s" % self.name, details=str(e)) current = results[0][1].get(self.name, []) modlist = [] if frozenset(self.values) != frozenset(current): if len(current) == 0: modlist = [(ldap.MOD_ADD, self.name, self.values)] elif len(self.values) == 0: modlist = [(ldap.MOD_DELETE, self.name, None)] else: modlist = [(ldap.MOD_REPLACE, self.name, self.values)] return modlist 

Example 16

    
    
    def _connect_to_ldap(self): ldap.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_NEVER) connection = ldap.initialize(self.server_uri) if self.start_tls: try: connection.start_tls_s() except ldap.LDAPError: e = get_exception() self.module.fail_json(msg="Cannot start TLS.", details=str(e)) try: if self.bind_dn is not None: connection.simple_bind_s(self.bind_dn, self.bind_pw) else: connection.sasl_interactive_bind_s('', ldap.sasl.external()) except ldap.LDAPError: e = get_exception() self.module.fail_json( msg="Cannot bind to the server.", details=str(e)) return connection 

Example 17

    
    
    def getMemberName(self, member): self.__assertIsMember(member) try: return self.__ldap_mail_to_cn(member) except ldap.LDAPError: raise NotAMemberError 

Example 18

    
    
    def set_password(self, username, hashes): """ Administratively set the user's password using the given hashes. """ dn = 'uid={0},{1}'.format(username, self.base_dn) try: with self._ldap_connection() as ldap_cxn: ldap_cxn.simple_bind_s(self.bind_dn, self.bind_pw) mod_nt = (ldap.MOD_REPLACE, 'sambaNTPassword', hashes['sambaNTPassword']) mod_ssha = (ldap.MOD_REPLACE, 'userPassword', hashes['userPassword']) mod_list = [mod_nt, mod_ssha] ldap_cxn.modify_s(dn, mod_list) except ldap.INVALID_CREDENTIALS: self.bus.log('Invalid credentials for admin user: {0}'.format(self.bind_dn), 40) raise except ldap.INSUFFICIENT_ACCESS: self.bus.log('Insufficient access for admin user: {0}'.format(self.bind_dn), 40) raise except ldap.INVALID_DN_SYNTAX: self.bus.log('Invalid DN syntax in configuration: {0}'.format(self.base_dn), 40) raise except ldap.LDAPError as e: self.bus.log('LDAP Error: {0}'.format(e.message['desc'] if 'desc' in e.message else str(e)), level=40, traceback=True) raise 

Example 19

    
    
    def change_password(self, username, oldpassword, hashes): """ Change the user's password using their own credentials. """ dn = 'uid={0},{1}'.format(username, self.base_dn) try: with self._ldap_connection() as ldap_cxn: ldap_cxn.simple_bind_s(dn, oldpassword) # don't use LDAPObject.passwd_s() here to make use of # ldap's atomic operations. IOW, don't change one password # but not the other. mod_nt = (ldap.MOD_REPLACE, 'sambaNTPassword', hashes['sambaNTPassword']) mod_ssha = (ldap.MOD_REPLACE, 'userPassword', hashes['userPassword']) mod_list = [mod_nt, mod_ssha] ldap_cxn.modify_s(dn, mod_list) except ldap.INVALID_CREDENTIALS: raise except ldap.INVALID_DN_SYNTAX: self.bus.log('Invalid DN syntax in configuration: {0}'.format(self.base_dn), 40) raise except ldap.LDAPError as e: self.bus.log('LDAP Error: {0}'.format(e.message['desc'] if 'desc' in e.message else str(e)), level=40, traceback=True) raise 

Example 20

    
    
    def userexistsbyuid(self, username): baseDN = "ou=People,"+self.dc searchScope = ldap.SCOPE_SUBTREE searchFilter = "uid="+username try: result = self.conn.search_s(baseDN, searchScope, searchFilter, None) if ( result==[] ): return False else: return True except ldap.LDAPError as e: print(e) return False 

Example 21

    
    
    def changepwd(self, username, newsecret): if (not self.userexistsbyuid(username) ): print("User %s does not exist!", username) return dn = "uid="+username+",ou=People,"+self.dc try: self.conn.passwd( dn, None, newsecret ) except ldap.LDAPError as e: print("Error: Can\'t change %s password: %s" % (username, e.message['desc'])) 

Example 22

    
    
    def changeshadowexpire(self, username, shexp): if (not self.userexistsbyuid(username)): print("User %s does not exist!", username) return dn = "uid="+username+",ou=People,"+self.dc ldif = [( ldap.MOD_REPLACE, 'shadowExpire', shexp )] try: self.conn.modify_s(dn, ldif) except ldap.LDAPError as e: print("Error: Can\'t change %s shadowExpire: %s" % (username, e.message['desc'])) 

Example 23

    
    
    def _ldap_function_call(func,*args,**kwargs): """ Wrapper function which locks calls to func with via module-wide ldap_lock """ if __debug__: if ldap._trace_level>=1: ldap._trace_file.write('*** %s.%s (%s,%s)\n' % ( '_ldap',repr(func), repr(args),repr(kwargs) )) if ldap._trace_level>=3: traceback.print_stack(limit=ldap._trace_stack_limit,file=ldap._trace_file) ldap._ldap_module_lock.acquire() try: try: result = func(*args,**kwargs) finally: ldap._ldap_module_lock.release() except LDAPError,e: if __debug__ and ldap._trace_level>=2: ldap._trace_file.write('=> LDAPError: %s\n' % (str(e))) raise if __debug__ and ldap._trace_level>=2: if result!=None and result!=(None,None): ldap._trace_file.write('=> result: %s\n' % (repr(result))) return result 

Example 24

    
    
    def _ldap_call(self,func,*args,**kwargs): """ Wrapper method mainly for serializing calls into OpenLDAP libs and trace logs """ if __debug__: if self._trace_level>=1:# and func.__name__!='result': self._trace_file.write('*** %s - %s (%s,%s)\n' % ( self._uri, self.__class__.__name__+'.'+func.__name__, repr(args),repr(kwargs) )) if self._trace_level>=3: traceback.print_stack(limit=self._trace_stack_limit,file=self._trace_file) self._ldap_object_lock.acquire() try: try: result = func(*args,**kwargs) if __debug__ and self._trace_level>=2: if func.__name__!="unbind_ext": diagnostic_message_success = self._l.get_option(ldap.OPT_DIAGNOSTIC_MESSAGE) else: diagnostic_message_success = None finally: self._ldap_object_lock.release() except LDAPError,e: if __debug__ and self._trace_level>=2: self._trace_file.write('=> LDAPError - %s: %s\n' % (e.__class__.__name__,str(e))) raise else: if __debug__ and self._trace_level>=2: if not diagnostic_message_success is None: self._trace_file.write('=> diagnosticMessage: %s\n' % (repr(diagnostic_message_success))) if result!=None and result!=(None,None): self._trace_file.write('=> result: %s\n' % (repr(result))) return result 

Example 25

    
    
    def _ldap_function_call(func,*args,**kwargs): """ Wrapper function which locks calls to func with via module-wide ldap_lock """ if __debug__: if ldap._trace_level>=1: ldap._trace_file.write('*** %s.%s (%s,%s)\n' % ( '_ldap',repr(func), repr(args),repr(kwargs) )) if ldap._trace_level>=3: traceback.print_stack(limit=ldap._trace_stack_limit,file=ldap._trace_file) ldap._ldap_module_lock.acquire() try: try: result = func(*args,**kwargs) finally: ldap._ldap_module_lock.release() except LDAPError,e: if __debug__ and ldap._trace_level>=2: ldap._trace_file.write('=> LDAPError: %s\n' % (str(e))) raise if __debug__ and ldap._trace_level>=2: if result!=None and result!=(None,None): ldap._trace_file.write('=> result: %s\n' % (repr(result))) return result 

Example 26

    
    
    def _ldap_call(self,func,*args,**kwargs): """ Wrapper method mainly for serializing calls into OpenLDAP libs and trace logs """ if __debug__: if self._trace_level>=1:# and func.__name__!='result': self._trace_file.write('*** %s - %s (%s,%s)\n' % ( self._uri, self.__class__.__name__+'.'+func.__name__, repr(args),repr(kwargs) )) if self._trace_level>=3: traceback.print_stack(limit=self._trace_stack_limit,file=self._trace_file) self._ldap_object_lock.acquire() try: try: result = func(*args,**kwargs) if __debug__ and self._trace_level>=2: if func.__name__!="unbind_ext": diagnostic_message_success = self._l.get_option(ldap.OPT_DIAGNOSTIC_MESSAGE) else: diagnostic_message_success = None finally: self._ldap_object_lock.release() except LDAPError,e: if __debug__ and self._trace_level>=2: self._trace_file.write('=> LDAPError - %s: %s\n' % (e.__class__.__name__,str(e))) raise else: if __debug__ and self._trace_level>=2: if not diagnostic_message_success is None: self._trace_file.write('=> diagnosticMessage: %s\n' % (repr(diagnostic_message_success))) if result!=None and result!=(None,None): self._trace_file.write('=> result: %s\n' % (repr(result))) return result 

Example 27

    
    
    def _ldap_function_call(lock,func,*args,**kwargs): """ Wrapper function which locks and logs calls to function lock Instance of threading.Lock or compatible func Function to call with arguments passed in via *args and **kwargs """ if lock: lock.acquire() if __debug__: if ldap._trace_level>=1: ldap._trace_file.write('*** %s.%s %s\n' % ( '_ldap',func.__name__, pprint.pformat((args,kwargs)) )) if ldap._trace_level>=9: traceback.print_stack(limit=ldap._trace_stack_limit,file=ldap._trace_file) try: try: result = func(*args,**kwargs) finally: if lock: lock.release() except LDAPError as e: if __debug__ and ldap._trace_level>=2: ldap._trace_file.write('=> LDAPError: %s\n' % (str(e))) raise if __debug__ and ldap._trace_level>=2: ldap._trace_file.write('=> result:\n%s\n' % (pprint.pformat(result))) return result 

Example 28

    
    
    def _ldap_call(self,func,*args,**kwargs): """ Wrapper method mainly for serializing calls into OpenLDAP libs and trace logs """ self._ldap_object_lock.acquire() if __debug__: if self._trace_level>=1: self._trace_file.write('*** %s %s - %s\n%s\n' % ( repr(self), self._uri, '.'.join((self.__class__.__name__,func.__name__)), pprint.pformat((args,kwargs)) )) if self._trace_level>=9: traceback.print_stack(limit=self._trace_stack_limit,file=self._trace_file) diagnostic_message_success = None try: try: result = func(*args,**kwargs) if __debug__ and self._trace_level>=2: if func.__name__!="unbind_ext": diagnostic_message_success = self._l.get_option(ldap.OPT_DIAGNOSTIC_MESSAGE) finally: self._ldap_object_lock.release() except LDAPError as e: if __debug__ and self._trace_level>=2: self._trace_file.write('=> LDAPError - %s: %s\n' % (e.__class__.__name__,str(e))) raise else: if __debug__ and self._trace_level>=2: if not diagnostic_message_success is None: self._trace_file.write('=> diagnosticMessage: %s\n' % (repr(diagnostic_message_success))) self._trace_file.write('=> result:\n%s\n' % (pprint.pformat(result))) return result 

Example 29

    
    
    def post(self): """ Queue the specific pipeline type """ user = request.form['user'] password = request.form['password'] # ip = request.remote_addr ut.pretty_print("Checking user %s" % user) # TODO : Provide proper code for special users wilee and demux if((cfg.ACME_DEV or cfg.ACME_PROD) and (user == 'wilee' or user == 'demux')): return {'token': generate_token(user, password)} else: try: # Make sure the password is not empty to prevent LDAP anonymous bind to succeed if not password: raise ldap.LDAPError("Empty password for user %s!" % user) #check if the LDAP binding works ldap.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_NEVER) conn = ldap.initialize(cfg.LDAPS_ADDRESS) conn.set_option(ldap.OPT_REFERRALS, 0) conn.set_option(ldap.OPT_PROTOCOL_VERSION, 3) conn.set_option(ldap.OPT_X_TLS_CACERTFILE, cfg.search_cfg_file('certificate_file.cer')) conn.set_option(ldap.OPT_X_TLS,ldap.OPT_X_TLS_DEMAND) conn.set_option(ldap.OPT_X_TLS_DEMAND, True) bind = conn.simple_bind_s("%[[email protected]](https://www.programcreek.com/cdn-cgi/l/email-protection)%s"%user, password, cfg.COMPANY_ADDRESS) ut.pretty_print("Bind success!!") #then create a session return {'token' : generate_token(user, password)} except ldap.LDAPError, e: ut.pretty_print("ERROR: authentication error: %s" % e) return "Authentication error", 401 

Example 30

    
    
    def ldap_get_vaild(self,uid=None,passwd=None): obj = self.ldapconn target_cn = self.ldap_search_dn(uid) try: if obj.simple_bind_s(target_cn,passwd): return True else: return False except ldap.LDAPError,e: print e #?????? 

Example 31

    
    
    def ldap_update_pass(self,uid=None,oldpass=None,newpass=None): modify_entry = [(ldap.MOD_REPLACE,'userpassword',newpass)] obj = self.ldapconn target_cn = self.ldap_search_dn(uid) try: obj.simple_bind_s(target_cn,oldpass) obj.passwd_s(target_cn,oldpass,newpass) return True except ldap.LDAPError,e: return False 

Example 32

    
    
    def ldap_search(module, conn, dn, search_filter, ldap_attrs): try: search = conn.search_s(dn, ldap.SCOPE_SUBTREE, search_filter, ldap_attrs) except ldap.LDAPError as e: fail_msg = "LDAP Error Searching: {}".format(ldap_errors(e)) module.fail_json(msg=fail_msg) return search 

Example 33

    
    
    def ldap_unbind(module, conn): result = False try: conn.unbind_s() result = True except ldap.LDAPError as e: fail_msg = "LDAP Error unbinding: {}".format(e) module.fail_json(msg=fail_msg) return result 

Example 34

    
    
    def do_bind(self): try: self.con.simple_bind_s(self.username, self.password) self.is_binded = True return True except ldap.INVALID_CREDENTIALS: print "[!] Error: invalid credentials" sys.exit(1) except ldap.LDAPError, e: print "[!] {}".format(e) sys.exit(1) 

Example 35

    
    
    def whoami(self): try: current_dn = self.con.whoami_s() except ldap.LDAPError, e: print "[!] {}".format(e) sys.exit(1) return current_dn 

Example 36

    
    
    def getAllUsers(self, attrs=''): if not attrs: attrs = ['cn', 'userPrincipalName'] objectFilter = '(objectCategory=user)' base_dn = self.domainBase try: rawUsers = self.do_ldap_query(base_dn, ldap.SCOPE_SUBTREE, objectFilter, attrs) except LDAPError, e: print "[!] Error retrieving users" print "[!] {}".format(e) sys.exit(1) return (self.get_search_results(rawUsers), attrs) 

Example 37

    
    
    def getAllGroups(self,attrs=''): 
      if not attrs: attrs = ['distinguishedName', 'cn'] 
      objectFilter = '(objectCategory=group)' 
      base_dn = self.domainBase 
      try: 
        rawGroups = self.do_ldap_query(base_dn, ldap.SCOPE_SUBTREE, objectFilter, attrs) 
      except LDAPError, e: 
        print "[!] Error retrieving groups" print "[!] {}".format(e) 
        sys.exit(1) 
        return (self.get_search_results(rawGroups), attrs) 

Example 38

    
    
    def doCustomSearch(self, base, objectFilter, attrs): try: rawResults = self.do_ldap_query(base, ldap.SCOPE_SUBTREE, objectFilter, attrs) except LDAPError, e: "print [!] Error doing search" "print [!] {}".format(e) sys.exit(1) return self.get_search_results(rawResults) 

Example 39

    
    
    def getAllComputers(self, attrs=''): if not attrs: attrs = ['cn', 'dNSHostName', 'operatingSystem', 'operatingSystemVersion', 'operatingSystemServicePack'] objectFilter = '(objectClass=Computer)' base_dn = self.domainBase try: rawComputers = self.do_ldap_query(base_dn, ldap.SCOPE_SUBTREE, objectFilter, attrs) except LDAPError, e: print "[!] Error retrieving computers" print "[!] {}".format(e) sys.exit(1) return (self.get_search_results(rawComputers), attrs) 

Example 40

    
    
    def ldap_search(self, filter, attributes, incremental, incremental_filter): """ Query the configured LDAP server with the provided search filter and attribute list. """ for uri in self.conf_LDAP_SYNC_BIND_URI: #Read record of this uri if (self.working_uri == uri): adldap_sync = self.working_adldap_sync created = False else: adldap_sync, created = ADldap_Sync.objects.get_or_create(ldap_sync_uri=uri) if ((adldap_sync.syncs_to_full > 0) and incremental): filter_to_use = incremental_filter.replace('?', self.whenchanged.strftime(self.conf_LDAP_SYNC_INCREMENTAL_TIMESTAMPFORMAT)) logger.debug("Using an incremental search. Filter is:'%s'" % filter_to_use) else: filter_to_use = filter ldap.set_option(ldap.OPT_REFERRALS, 0) #ldap.set_option(ldap.OPT_NETWORK_TIMEOUT, 10) l = PagedLDAPObject(uri) l.protocol_version = 3 if (uri.startswith('ldaps:')): l.set_option(ldap.OPT_X_TLS, ldap.OPT_X_TLS_DEMAND) l.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_DEMAND) l.set_option(ldap.OPT_X_TLS_DEMAND, True) else: l.set_option(ldap.OPT_X_TLS, ldap.OPT_X_TLS_NEVER) l.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_NEVER) l.set_option(ldap.OPT_X_TLS_DEMAND, False) try: l.simple_bind_s(self.conf_LDAP_SYNC_BIND_DN, self.conf_LDAP_SYNC_BIND_PASS) except ldap.LDAPError as e: logger.error("Error connecting to LDAP server %s : %s" % (uri, e)) continue results = l.paged_search_ext_s(self.conf_LDAP_SYNC_BIND_SEARCH, ldap.SCOPE_SUBTREE, filter_to_use, attrlist=attributes, serverctrls=None) l.unbind_s() if (self.working_uri is None): self.working_uri = uri self.conf_LDAP_SYNC_BIND_URI.insert(0, uri) self.working_adldap_sync = adldap_sync return (uri, results) # Return both the LDAP server URI used and the request. This is for incremental sync purposes #if not connected correctly, raise error raise 

Example 41

    
    
    def authenticate(login, password): """ Attempt to authenticate the login name with password against the configured LDAP server. If the user is authenticated, required group memberships are also verified. """ lconn = open_ldap() server = _config.get('ldap', 'server') user = LDAPUser(login, lconn) # Bind to user using the supplied password try: user.bind(password) except (ldap.SERVER_DOWN, ldap.CONNECT_ERROR): _logger.exception("LDAP server is down") raise NoAnswerError(server) except ldap.INVALID_CREDENTIALS: _logger.warning("Server %s reported invalid credentials for user %s", server, login) return False except ldap.TIMEOUT as error: _logger.error("Timed out waiting for LDAP bind operation") raise TimeoutError(error) except ldap.LDAPError: _logger.exception("An LDAP error occurred when authenticating user %s " "against server %s", login, server) return False except UserNotFound: _logger.exception("Username %s was not found in the LDAP catalog %s", login, server) return False _logger.debug("LDAP authenticated user %s", login) # If successful so far, verify required group memberships before # the final verdict is made group_dn = _config.get('ldap', 'require_group') if group_dn: if user.is_group_member(group_dn): _logger.info("%s is verified to be a member of %s", login, group_dn) return user else: _logger.warning("Could NOT verify %s as a member of %s", login, group_dn) return False # If no group matching was needed, we are already authenticated, # so return that. return user 

Example 42

    
    
    def main(): """Program entry function.""" # XXX: Stupid Apache on shrapnel has TZ set to US/Eastern, no idea why! os.environ['TZ'] = 'Eire' print("Content-type: text/html") print() atexit.register(shutdown) # Sets up an exception handler for uncaught exceptions and saves # traceback information locally. # cgitb.enable(logdir='%s/tracebacks' % os.getcwd()) global form form = cgi.FieldStorage() opt.mode = form.getfirst('mode') if opt.mode not in cmds: opt.mode = 'card' opt.action = form.getfirst('action') # XXX remove usr.override # usr.override = opt.override = form.getfirst('override') == '1' opt.override = form.getfirst('override') == '1' # Start HTML now only for modes that print output *before* html_form is # called (which calls start_html itself). We delay the printing of the # header for all other modes as mode switching may occur (e.g. # cardid <-> add/renew). # if opt.mode in cmds_noform or (opt.mode in cmds_custom and opt.action): html_start() global udb udb = RBUserDB() udb.setopt(opt) # Open database and call function for specific command only if action # is required or the command needs no user input (i.e. no blank form # stage). # if opt.mode in cmds_noform or opt.action: try: udb.connect() except ldap.LDAPError as err: error(err, 'Could not connect to user database') # not reached try: eval(opt.mode + '()') except (ldap.LDAPError, RBError) as err: error(err) # not reached html_form() sys.exit(0) 
