grpcurl -plaintext -d '{
"email":"test@example.com",
"subject":"SMTP Test",
"body":"test 2"
}' localhost:50051 notification.NotificationService.SendEmail
{
"success": true,
"message": "Email queued"
}


grpcurl -plaintext -d '{
"videoId":"123",
"email":"test@example.com",
"filename":"video.mp4",
"status":"completed"
}' localhost:50051 notification.NotificationService.SendProcessingCompleted