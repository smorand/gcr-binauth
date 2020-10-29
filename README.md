# Test of Google Cloud Run Binary authorization

## Prepare

At first, position the project and project number:
```
export PROJECT=$(gcloud config get-value core/project)
export PROJECT_NUMBER=PROJECT_NUMBER=$(gcloud projects list \
    --filter="${PROJECT}" --format="value(PROJECT_NUMBER)")
export NOTE_ID=me
```

Build also a specific image to speed up cloud run deployment:
```
cd builder_cloud-sdk-beta
gcloud builds submit -t gcr.io/$(gcloud config get-value project)/cloud-sdk-beta
```

1. Create KMS credentials keyring
```
gcloud kms keyrings create binauth \
    --project ${PROJECT} \
    --location europe-west1
```

2. Create KMS credentials keys
```
gcloud kms keys create binauth \
    --project ${PROJECT} \
    --keyring binauth \
    --location europe-west1 \
    --purpose=asymmetric-signing \
    --default-algorithm rsa-sign-pss-4096-sha256
```

3. Add permission to use the key
```
gcloud kms keys add-iam-policy-binding binauth \
    --project ${PROJECT} \
    --location europe-west1 \
    --keyring binauth \
    --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
    --role roles/cloudkms.signerVerifier
```

4. Add permissions to view ancestors
```
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/binaryauthorization.attestorsViewer
```

5. Add permissions to attach notes
```
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/containeranalysis.notes.attacher
```

6. Create an ancestor
Refer to:
```
cat | curl -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    --data-binary @-  \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT}/notes/?noteId=${NOTE_ID}" <<EOF
{
  "name": "projects/${PROJECT}/notes/${NOTE_ID}",
  "attestation": {
    "hint": {
      "human_readable_name": "This note was auto-generated for attestor ${NOTE_ID}"
    }
  }
}
EOF

gcloud  beta container binauthz attestors create "${NOTE_ID}" \
    --project="${PROJECT}" \
    --attestation-authority-note="${NOTE_ID}" \
    --attestation-authority-note-project="${PROJECT}"
```

7. Attach the KMS key to the attestors
```
gcloud container binauthz attestors public-keys add \
    --project="${PROJECT}" \
    --attestor me \
    --keyversion-project ${PROJECT} \
    --keyversion-location europe-west1 \
    --keyversion-keyring binauth \
    --keyversion-key binauth \
    --keyversion 1
```


## Test

1. Deploy the cloud run:
```
gcloud build submit
```

Et check the result:
```
curl -s -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" https://helloworld-qem5z53dna-ew.a.run.app
```

Actually as long as it deploy. You can try if you remove the step "create-attestation" in cloud build
it won't deploy anymore.