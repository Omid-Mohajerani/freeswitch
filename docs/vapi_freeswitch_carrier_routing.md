# Connect VAPI BYO SIP Trunk to FreeSWITCH and Route to Any Carrier

This guide shows how to:
1. Create a Bring Your Own (BYO) SIP trunk in VAPI.
2. Assign a phone number to that trunk.
3. Configure FreeSWITCH to handle calls from VAPI.
4. Route calls from VAPI to any carrier (example shown).

---

🎥 **Video Tutorial**  
You can also watch the full step-by-step video here: [Connect VAPI BYO SIP Trunk to FreeSWITCH](https://www.youtube.com/watch?v=HkWcpuMrX7k&t=2s)

---



## 1. Create the VAPI SIP Trunk

Use the `POST /credential` endpoint to create a BYO SIP trunk in VAPI.

```bash
curl -X POST "https://api.vapi.ai/credential" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer YOUR_VAPI_API_KEY" \\
  -d '{
    "provider": "byo-sip-trunk",
    "name": "FS01",
    "gateways": [{
      "ip": "YOUR_FREESWITCH_PUBLIC_IP",
      "inboundEnabled": true
    }],
    "outboundLeadingPlusEnabled": true,
    "outboundAuthenticationPlan": {
      "authUsername": "YOUR_SIP_USERNAME",
      "authPassword": "YOUR_SIP_PASSWORD"
    }
  }'
```

### **Example**
```bash
curl -X POST "https://api.vapi.ai/credential" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer 12213131-121212-4c3cf-8aee-e7cc4ac29a0b" \\
  -d '{
    "provider": "byo-sip-trunk",
    "name": "FS01",
    "gateways": [{
      "ip": "138.68.68.195",
      "inboundEnabled": true
    }],
    "outboundLeadingPlusEnabled": true,
    "outboundAuthenticationPlan": {
      "authUsername": "hVbqYkTnPzRmLaXoWdEcJufIsg",
      "authPassword": "pXnVhGqZmRjLtSaYwKeDbCoUfi"
    }
  }'
```

### **Sample Output**
```json
{
  "id": "b2b51f47-9999e-4e02-b737-bcfdc2cfde5e",
  "orgId": "d9961a62-11c1-4991-984c-e83c6ca07eaa",
  "provider": "byo-sip-trunk",
  "createdAt": "2025-08-15T12:19:16.171Z",
  "updatedAt": "2025-08-15T12:19:16.171Z",
  "gateways": [
    {
      "ip": "138.68.68.195",
      "inboundEnabled": true
    }
  ],
  "name": "FS01",
  "outboundAuthenticationPlan": {
    "authUsername": "hVbqYkTnPzRmLaXoWdEcJufIsg"
  },
  "outboundLeadingPlusEnabled": true
}
```

---

## 2. Assign a Number to the Trunk

Use the `POST /phone-number` endpoint to assign your inbound number to the trunk.

```bash
curl -X POST "https://api.vapi.ai/phone-number" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer YOUR_VAPI_API_KEY" \\
  -d '{
    "provider": "byo-phone-number",
    "name": "Number +4915223344556",
    "number": "4915223344556",
    "numberE164CheckEnabled": false,
    "credentialId": "YOUR_TRUNK_ID"
  }'
```

### **Example**
```bash
curl -X POST "https://api.vapi.ai/phone-number" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer d23232323-d011-4ccf-2323-e7c33ac29a0b" \\
  -d '{
    "provider": "byo-phone-number",
    "name": "Number +4915223344556",
    "number": "4915223344556",
    "numberE164CheckEnabled": false,
    "credentialId": "b2b51f47-9999e-4e02-b737-bcfdc2cfde5e"
  }'
```

### **Sample Output**
```json
{
  "id": "554ae397-b61f-4c80-9f68-cb0f79349ba6",
  "orgId": "d8861a62-a5c1-4991-984c-e83c64a07555",
  "number": "4915223344556",
  "createdAt": "2025-08-15T12:20:48.249Z",
  "updatedAt": "2025-08-15T12:20:48.249Z",
  "name": "Number +4915223344556",
  "credentialId": "b2b51f47-9999e-4e02-b737-bcfdc2cfde5e",
  "provider": "byo-phone-number",
  "numberE164CheckEnabled": false,
  "status": "active",
  "providerResourceId": "de50c6f3-dc82-4d70-8cf5-d232323a41a1261"
}
```

---

## 3. Configure FreeSWITCH Directory

Create a SIP user in FreeSWITCH to match the trunk credentials.

**`/etc/freeswitch/directory/default/vapi.xml`**
```xml
<include>
  <user id="hVbqYkTnPzRmLaXoWdEcJufIsg">
    <params>
      <param name="password" value="pXnVhGqZmRjLtSaYwKeDbCoUfi"/>
    </params>
    <variables>
      <variable name="user_context" value="vapicontext"/>
      <variable name="outbound_caller_id_name" value="$${outbound_caller_name}"/>
      <variable name="outbound_caller_id_number" value="$${outbound_caller_id}"/>
    </variables>
  </user>
</include>
```

---

## 4. Create the `vapicontext` Dialplan

This context handles inbound calls from VAPI and routes them to your chosen carrier.

**`/etc/freeswitch/dialplan/vapicontext.xml`**
```xml
<include>
  <context name="vapicontext">

    <!-- Route German E.164 numbers (+49...) -->
    <extension name="from-vapi-to-carrier-de">
      <condition field="destination_number" expression="^\\+49\\d+$">
        <action application="set" data="hangup_after_bridge=true"/>
        <action application="bridge" data="sofia/external/${destination_number}@carrier.example.com"/>
      </condition>
    </extension>

    <!-- Route US E.164 numbers (+1...) -->
    <extension name="from-vapi-to-carrier-us">
      <condition field="destination_number" expression="^\\+1\\d+$">
        <action application="set" data="hangup_after_bridge=true"/>
        <action application="bridge" data="sofia/external/${destination_number}@carrier.example.com"/>
      </condition>
    </extension>

    <!-- Fallback: return 404 if nothing matched -->
    <extension name="no-route">
      <condition field="destination_number" expression=".*">
        <action application="respond" data="404 Not Found"/>
        <action application="hangup" data="NO_ROUTE_DESTINATION"/>
      </condition>
    </extension>

  </context>
</include>
```

---

## 5. Reload Configuration

After creating the XML files:

```bash
fs_cli -x 'reloadxml'
fs_cli -x 'sofia profile external rescan'
```

---

## 6. Testing

1. Call the number assigned in Step 2.  
2. FreeSWITCH will receive the call in `vapicontext`.  
3. If the `destination_number` matches your regex, the call will be bridged to your carrier.


