Public Sites: setup => Sites -> fill required Info, Activate,-> give apex access to Guest user 

EndPint URL : https://helensdaughters--dev.sandbox.my.salesforce-sites.com/services/apexrest/zoom/webhook

Event :1) Webinar Created, Updated, deleted
       2) Webinar registration Created, Approved, Cancel


@RestResource(urlMapping='/zoom/webhook')
global without sharing class ZoomWebhookHandler {

    private static final String ZOOM_SECRET_TOKEN = '0IRWnbeaSOGUWDN-Kh1dmg';

    @HttpPost
    global static void handleWebhook() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;
        if (res == null) {
            res = new RestResponse();
            RestContext.response = res;
        }
        res.statusCode = 200;

        String body = req.requestBody == null ? '' : req.requestBody.toString();

        Map<String, Object> data;
        try {
            data = (Map<String, Object>) JSON.deserializeUntyped(body);
        } catch (Exception e) {
            res.responseBody = Blob.valueOf('{"error":"Invalid JSON"}');
            return;
        }

        if (data == null || !data.containsKey('event')) {
            res.responseBody = Blob.valueOf('{}');
            return;
        }

        String eventType = (String) data.get('event');

                if (eventType == 'endpoint.url_validation') {
                    Map<String, Object> payload = (Map<String, Object>) data.get('payload');
                    String plainToken = (String) payload.get('plainToken');
                
                    Blob key = Blob.valueOf(ZOOM_SECRET_TOKEN);
                    Blob dataBlob = Blob.valueOf(plainToken);
                
                    Blob mac = Crypto.generateMac('HmacSHA256', dataBlob, key);
                    String encryptedToken = EncodingUtil.convertToHex(mac);
                
                    res.statusCode = 200;
                    res.responseBody = Blob.valueOf(
                        JSON.serialize(new Map<String, String>{
                            'plainToken' => plainToken,
                            'encryptedToken' => encryptedToken
                        })
                    );
                    return;
                }
                
        try {
            if (eventType == 'webinar.created') {
                    handleWebinarCreated(data);
                }
                else if (
                    eventType == 'webinar.registration_created' ||
                    eventType == 'webinar.registration_approved'
                ) {
                    handleRegistrantCreated(data);
                }


            res.responseBody = Blob.valueOf('{"status":"success"}');

        } catch (Exception e) {
            System.debug('Error processing Zoom webhook: ' + e.getMessage());
            res.statusCode = 500;
            res.responseBody = Blob.valueOf('{"error":"Internal server error"}');
        }
    }

    private static void handleWebinarCreated(Map<String, Object> data) {
        Map<String, Object> payload = (Map<String, Object>) data.get('payload');
        Map<String, Object> obj = (Map<String, Object>) payload.get('object');

        String webinarId = String.valueOf(obj.get('id'));

        List<Program> existing = [
            SELECT Id FROM Program WHERE Zoom_Webinar_Id__c = :webinarId LIMIT 1
        ];
        if (!existing.isEmpty()) return;

        String startTimeStr = (String) obj.get('start_time');
        Date startDate = null;
        if (startTimeStr != null && startTimeStr.length() >= 10) {
            startDate = Date.valueOf(startTimeStr.substring(0, 10));
        }

        Program prog = new Program(
            Name = (String) obj.get('topic'),
            Program_Type__c = 'Ag-cademy',
            StartDate = startDate,
            Zoom_Webinar_Id__c = webinarId
        );
        insert prog;
        System.debug('Created Program: ' + prog.Id + ' for Webinar: ' + webinarId);
    }

   private static void handleRegistrantCreated(Map<String, Object> data) {

    Map<String, Object> payload = (Map<String, Object>) data.get('payload');
    Map<String, Object> obj = (Map<String, Object>) payload.get('object');

    // Webinar Id is always here
    String webinarId = String.valueOf(obj.get('id'));

    // 🔥 Handle payload difference
    Map<String, Object> registrant;
    if (obj.containsKey('registrant')) {
        registrant = (Map<String, Object>) obj.get('registrant');
    } else {
        registrant = obj;
    }

    String email = (String) registrant.get('email');
    String firstName = (String) registrant.get('first_name');
    String lastName  = (String) registrant.get('last_name');

    System.debug('webinarId=' + webinarId);
    System.debug('email=' + email);

    if (String.isBlank(email)) {
        System.debug('Email missing in payload, skipping');
        return;
    }

    List<Program> programs = [
        SELECT Id FROM Program WHERE Zoom_Webinar_Id__c = :webinarId LIMIT 1
    ];
    if (programs.isEmpty()) return;

    Program prog = programs[0];

    Contact con;
    List<Contact> contacts = [
        SELECT Id FROM Contact WHERE Email = :email LIMIT 1
    ];
    if (contacts.isEmpty()) {
        con = new Contact(
            FirstName = firstName,
            LastName  = lastName,
            Email     = email
        );
        insert con;
    } else {
        con = contacts[0];
    }

    List<ProgramEnrollment> existing = [
        SELECT Id FROM ProgramEnrollment
        WHERE ProgramId = :prog.Id AND ContactId = :con.Id LIMIT 1
    ];
    if (!existing.isEmpty()) return;

    insert new ProgramEnrollment(
        Name = firstName + ' ' + lastName,
        ProgramId = prog.Id,
        ContactId = con.Id
    );

    System.debug('ProgramEnrollment created successfully');
}

}
